---
layout: post
title: "Main函数之前的那些事儿02：App启动速度优化"
date: 2016-05-10 13:36
categories: 
- 技术
- [技术, iOS]
tags: 
- 技术
- iOS
---
> **公式：App总启动时间 = main()函数之前的加载时间 + main()之后的加载时间**
>
> - main()之前的加载时间：dylib链接动态库的时间+自身App可执行文件的加载时间
> - main()之后的加载时间：main方法执行之后到AppDelegate的didFinishLaunchingWithOptions方法执行结束前这段时间，主要是构建第一个界面，并完成渲染展示。

## 分析

尽量别在+load方法里搞事情，且对实现了+load()方法的类进行分析，尽量将load里的代码延后调用。

### **+load优化思路**

<!-- more -->

* 在load方法中需要执行的代码通过block的方式添加进一个全局的数组中，然后在对应的时刻，执行该block。
* 加载时机（分为3个时刻）
  1. 系统调用完所有class的load方法之后，且在main函数执行之前。（__attribute__((constructor))）
  2. 程序didFinishLaunch。（UIApplicationDidBecomeActiveNotification通知）
  3. 程序didFinishLaunch之后。（在2之后延迟5s）
* 结合GCD实现同步异步执行
  有些代码需要在主线程同步执行，有的则可以异步去执行。（提供2种类型的接口）
* 优先级问题
  同步执行的过程中，有的class的load方法要在其他的class的load方法之前去执行，就需要一个优先级控制。（通过参数，数组排序实现）

## **demo**

```
//
//  XRLoadManager.h
//  xr_load
//
//  Created by ARui on 2017/4/19.
//  Copyright © 2017年 XR. All rights reserved.
//

#import <Foundation/Foundation.h>
typedef NS_ENUM(NSUInteger, LoadAt) {
    /**
     *  准备执行main函数，优先级最高
     */
    LoadAtPreMain = 0,
    /**
     *  应用进入活跃状态, 在PreMain之后
     */
    LoadAtAppLaunched,
    /**
     *  遥远的时间, 大致在 becomeActive 执行完后 5s
     */
    LoadAtIdleTime,
    
    /////
    LoadAtN
};

@interface XRLoadManager : NSObject

+ (void)load_async:(LoadAt)at block:(void(^)(void))block;
+ (void)load_main:(LoadAt)at block:(void(^)(void))block;
+ (void)load_main:(LoadAt)at sort:(NSInteger)sort block:(void(^)(void))block;

@end

```

```
//
//  XRLoadManager.m
//  xr_load
//
//  Created by ARui on 2017/4/19.
//  Copyright © 2017年 XR. All rights reserved.
//

#import "XRLoadManager.h"
#import <UIKit/UIKit.h>
#import <Foundation/Foundation.h>

@interface _XRLoadObject : NSObject
@property (nonatomic, assign) LoadAt at;
@property (nonatomic, assign) NSInteger sort;
@property (nonatomic, assign) BOOL mainThread;
@property (nonatomic, copy) void (^block) (void);
@end

@implementation _XRLoadObject

@end

@implementation XRLoadManager
static NSMutableArray *loadables[LoadAtN];


+ (void)load_init {
    static dispatch_once_t once;
    dispatch_once(&once, ^{
        for (int i = 0; i < LoadAtN; i++) {
            loadables[i] = [NSMutableArray array];
        }
    });
    
    [[NSNotificationCenter defaultCenter] addObserver:self selector:@selector(handleAppLaunched:) name:UIApplicationDidBecomeActiveNotification object:nil];
}

+ (void)load_deInit {
    for (int i = 0; i < LoadAtN; i++) {
        NSMutableArray *array = loadables[i];
        [array removeAllObjects];
    }
}

+ (void)handleAppLaunched:(NSNotification *)note {
    [[NSNotificationCenter defaultCenter] removeObserver:self];
    
    [self run_launched];
    
    // run idle
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 5 * NSEC_PER_SEC), dispatch_get_global_queue(0, 0), ^{
        [self run_idleTime];
        [self load_deInit];
    });
}

+ (void)load_async:(LoadAt)at block:(void(^)(void))block {
    [self load_some:at sort:100 mainthread:NO block:block];
}

+ (void)load_main:(LoadAt)at block:(void(^)(void))block {
    [self load_some:at sort:100 mainthread:YES block:block];
}

+ (void)load_main:(LoadAt)at sort:(NSInteger)sort block:(void(^)(void))block {
    [self load_some:at sort:sort mainthread:YES block:block];
}

+ (void)load_some:(LoadAt)at sort:(NSInteger)sort mainthread:(BOOL)mainthread block:(void(^)(void))block {
    assert(block);
    _XRLoadObject *obj = [[_XRLoadObject alloc] init];
    obj.at = at;
    obj.sort = sort;
    obj.mainThread = mainthread;
    obj.block = block;
    [self load_init];
    [loadables[at] addObject:obj];
}

+ (void)run_preMain {
    [self run_block_array:LoadAtPreMain];
}

+ (void)run_launched {
    [self run_block_array:LoadAtAppLaunched];
}

+ (void)run_idleTime {
    [self run_block_array:LoadAtIdleTime];
}

+ (void)run_block_array:(LoadAt)at {
    
    void(^sortLoadBlock)(NSMutableArray *array) = ^(NSMutableArray *array) {
        [array sortUsingComparator:^NSComparisonResult(_XRLoadObject* _Nonnull obj1, _XRLoadObject* _Nonnull obj2) {
            if (!obj1.mainThread && obj2.mainThread) {
                return NSOrderedDescending;
            }
            if (obj1.mainThread && !obj2.mainThread) {
                return NSOrderedAscending;
            }
            if (obj1.sort > obj2.sort) {
                return NSOrderedDescending;
            }
            if (obj1.sort == obj2.sort) {
                return NSOrderedSame;
            }
            return NSOrderedAscending;
        }];
        for (_XRLoadObject *obj in array) {
            if (obj.mainThread) {
                if ([NSThread isMainThread]) {
                    obj.block();
                } else {
                    dispatch_async(dispatch_get_main_queue(), obj.block);
                }
            } else {
                dispatch_async(dispatch_get_global_queue(0, 0), obj.block);
            }
        }
    };
    if (at == LoadAtPreMain) {
        sortLoadBlock(loadables[LoadAtPreMain]);
    }else if (at == LoadAtAppLaunched) {
        sortLoadBlock(loadables[LoadAtAppLaunched]);
    }else if (at == LoadAtIdleTime) {
        sortLoadBlock(loadables[LoadAtIdleTime]);
    }
    
}

@end

__attribute__((constructor))
static void initlize(void) {
    [XRLoadManager run_preMain];
}
```
