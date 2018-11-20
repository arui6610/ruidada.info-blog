---
layout: post
title: "日常开发02：砸壳+Hook"
date: 2017-03-10 17:46
categories: 
- 技术
- [技术, iOS]
tags: 
- 技术
- iOS
- 日常
---
> 项目中或多或少都需要接入一些三方sdk，那种静态库。然而产品就经常会提出那种无理的需求，让我们去改sdk的东西。要知道去改一个不知道源码的东西，还要在不影响原有功能的情况下去插入自己的逻辑在里面。是件很危险的事情。所以...

![ed372f2eb9389b50f7c73c348c35e5dde6116e87.gif](http://upload-images.jianshu.io/upload_images/1488967-bdc0754b608af099.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

<!-- more -->

首先想到的就是利用切面编程去hook静态库里的某些方法。可是某些方法是指哪些方法？



![N7Z0}{DL4OEU4(3S`($1UH7.gif](http://upload-images.jianshu.io/upload_images/1488967-81bcd73464d974bc.gif?imageMogr2/auto-orient/strip)

---

## 砸壳

> 《iOS应用逆向工程》书中有提到一个工具(class-dump)，这个工具利用Objective-C的runtime特性，将存储在Mach-O文件中的头文件信息提取出来，并生成对应的.h文件。这样就能扒出来所有的头文件了。

### 使用教程

* 1.安装class-dump.dmg。

* 2.将class-dump复制到'/usr/bin'目录下。

  ```
  sudo cp /Users/xxx/Desktop/class-dump '/usr/bin'
  ```

* 3.拿出你要扒的ipa包解压，得到.app文件。

* 4.class-dump -H test.app -o testFile

  ![屏幕快照 2016-09-07 下午12.09.24.png](http://upload-images.jianshu.io/upload_images/1488967-bbcfdc45bc1df7a4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

OK。接下来你看到桌面上出现一个文件夹![屏幕快照 2016-09-07 下午12.11.23.png](http://upload-images.jianshu.io/upload_images/1488967-151d981f3b434cce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这么多头文件？小手一抖，点开看看吧。![屏幕快照 2016-09-07 下午12.12.05.png](http://upload-images.jianshu.io/upload_images/1488967-fbd09b88b9bb337c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你妹啊，近4000个头文件。

好吧。剩下的就是在这4000份文件里，找到你要hook的那个类，那个方法。
![D6E8D73FD503854EAD8084AB3298C4BC.jpg](http://upload-images.jianshu.io/upload_images/1488967-ae60e1453435432f.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

经过多次尝试，总结了一些找方法的经验。

* 1.断点，查看调用栈，根据调用栈判断最有可能的那个方法，从而缩小范围。
* 2.配合runtime，打印class的所有方法名，属性的key和value。
* 3.有时候，你hook的了某个类的某个方法，发现未生效，可能是该方法是继承与父类的，且自己没有重载父类的方法。这个时候需要hook父类的该方法。

---

## HOOK

### 方法hook

在oc中每个方法对应的都是一个sel和imp的映射，方法A的imp指针和方法B的imp指针进行了替换，并且在要替换的方法中调用了一下原来的方法实现。这个也是常用到的方法混淆。

例如：

```
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        Class class = [self class];
        
        SEL originalSelector = @selector(test);
        SEL swizzledSelector = @selector(swizzle_test);
        
        
        Method originalMethod = class_getInstanceMethod(class, originalSelector);
        Method swizzledMethod = class_getInstanceMethod(class, swizzledSelector);
        

        BOOL didAddMethod = class_addMethod(class, originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod));
        
        if (didAddMethod) {
            class_replaceMethod(class,
                                swizzledSelector,
                                method_getImplementation(originalMethod),
                                method_getTypeEncoding(originalMethod));
        } else {
            method_exchangeImplementations(originalMethod, swizzledMethod);
        }
    });
}

- (void)test {
    NSLog(@"test");
}
- (void)swizzle_test {
    [self swizzle_test];
}
```



### 函数hook

推荐使用[fishhook](https://github.com/facebook/fishhook )，使用起来很简单，内部实现复杂，很强，源码基本看不懂。大致原理还是要弄清楚的。

例如有2个函数，函数A，函数B，想要将函数A的实现替换为函数B的实现。fishhook的实现大致分为2步，第一步找到目标函数的函数地址。第二步找到函数名，进行比对。匹配成功，替换目标函数地址为自己的函数地址。

```
struct rebinding {
    const char *name; // 目标函数名
    void *replacement; // 替换函数
    void **replaced; // 二级目标函数指针
};
/*
  struct rebinding rebindings[]:要重新绑定的结构体数组
  rebindings_nel：要重新绑定的函数个数
*/
int rebind_symbols(struct rebinding rebindings[], size_t rebindings_nel);

int rebind_symbols_image(void *header,
                         intptr_t slide,
                         struct rebinding rebindings[],
                         size_t rebindings_nel);
```



#### 例1

hook Foundation库的NSSetUncaughtExceptionHandler()函数（生效）

```
static NSUncaughtExceptionHandler *g_vaildUncaughtExceptionHandler;
static void (*ori_NSSetUncaughtExceptionHandler)( NSUncaughtExceptionHandler *_Nullable);

void swizzle_NSSetUncaughtExceptionHandler( NSUncaughtExceptionHandler * handler) {
    g_vaildUncaughtExceptionHandler = NSGetUncaughtExceptionHandler();
    if (g_vaildUncaughtExceptionHandler != NULL) {
        NSLog(@"UncaughtExceptionHandler=%p",g_vaildUncaughtExceptionHandler);
    }
    ori_NSSetUncaughtExceptionHandler(handler);
    g_vaildUncaughtExceptionHandler = NSGetUncaughtExceptionHandler();
}

static void xr_uncaught_exception_handler (NSException *exception) {
    NSLog(@"XX");
}

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        rebind_symbols((struct rebinding [1]){"NSSetUncaughtExceptionHandler",swizzle_NSSetUncaughtExceptionHandler,(void *)&ori_NSSetUncaughtExceptionHandler}, 1);
        NSSetUncaughtExceptionHandler(&xr_uncaught_exception_handler);
         [@[] objectAtIndex:1];
        
    }
    return 0;
}
```



#### 例2

hook自己的函数（不生效）

```
void test(void) {
    NSLog(@"test");
}
void swizzle_test(void) {
    NSLog(@"swizzle_test");
}
static void (*funcptr)(void);

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        
        // 初始化一个 rebinding 结构体
        struct rebinding test_rebinding = { "test", swizzle_test, (void *)&funcptr };
        // 将结构体包装成数组，并传入数组的大小，对原符号 open 进行重绑定
        rebind_symbols((struct rebinding[1]){test_rebinding}, 1);
        // 调用原函数
        test();
        
    }
    return 0;
}
```



函数的hook大致原理和方法hook类似。方法调用其实在运行时也就是转成一个个函数调用。根据方法名（SEL）去查找对应的方法实现的函数地址(IMP)，然后执行该函数。只不过函数和方法的存储空间不一样，查找函数地址的实现方法也就不一样。

示例2中，我发现hook不生效。而示例1中可以。就很奇怪。查了资料，发现原来fishhook不能hook当前镜像（库或可执行文件）中运行的函数，当前镜像中的函数调用是直接通过函数地址调用。而其他镜像中的函数，是通过可执行文件中的符号化表去查找对应的函数地址，从而间接调用的函数。而fishhook就是在符号化表中做的手脚。那就能解释了实例2不生效的原因了。