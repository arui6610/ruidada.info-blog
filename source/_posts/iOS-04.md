---
layout: post
title: "AtomSDK分享（安全模式及crash保护）"
date: 2017-08-19 20:44
categories: 
- 技术
- [技术, iOS]
tags: 
- 技术
- iOS
---
![Atom.jpg](http://upload-images.jianshu.io/upload_images/1488967-4b6a2f23f722bb43.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/450)

> 前言：
> APP的Crash率一直是评估app稳定性的一个重要指标，也是程序员的噩梦。每次提测之前跑完用例，都是信誓旦旦的"放心，代码不可能存在bug。"。在iOS还存在热补丁的时候，大部分iOS程序狗都有过半夜写着JSPatch代码的体验，那个时候确实可以做到不存在bug。然而热补丁被苹果大大封杀了，我们就开始焦虑了。KPI怎么搞？问题来了，如果App出现不可挽救的闪退怎么办？比如启动就崩溃，随便点点就闪退。怎么办？怎么办？怎么办？

<!-- more -->

## 开发目的

主要的动机有2点。

一是为了防止app在启动时期crash的情况发生，因为这种crash一旦产生必然会使得app陷入无法启动的绝境。

另一个是为了降低运行时期app的crash率，主要是利用oc的动态特性，通过切面编程，使得业务层在无感知的情况下，app能够在运行时期自动捕获异常，修复异常，从而避免crash的出现。

## 功能简介

> 功能主要分为两个模块

### 安全模式

冷启动时期内程序能够检测连续闪退，在出现连续闪退时，能够进行自我修复，一共分为三级保护，触发一级保护时会删除沙盒中的一些网络数据的缓存。二级保护则会删除沙盒中的rn包以及一些资源文件。三级保护会去清空沙盒，从而使得app进入一种初始化状态。

### 进程守护

* 1.unrecognized selector crash
* 2.KVO crash
* 3.NSNotification crash
* 4.NSTimer crash
* 5.Container crash（常见集合类crash等）
* 6.NSNull crash 
* 7.Bad Access crash （野指针）
* 8.UI not on Main Thread Crash (非主线程刷UI）

### 日志上报

## 技术实现

### 安全模式

app在启动时期会执行一系列的初始化操作，例如：

* 协议注册
 * 配置接口
 * RN加载
 * 相关配置信息读取，环境变量配置
 * 本地数据缓存读取
 * ......


实现逻辑：
![安全模式.png](http://upload-images.jianshu.io/upload_images/1488967-3afff57c83997764.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1、开始检测
一般来说，应用启动一般是在appDelegate里的didFinishLaunchingWithOptions方法回调里搞事情。
可是发现自家的写代码喜欢在+load方法里搞。且+load方法的执行是在main函数之前，那么问题就来了， 无法保证+load方法里产生的一些异常也能够被检测到。为了避免这种情况，需要把安全检测的逻辑放在+load方法里，且被系统调度的优先级比较高。好在，+load方法是有过处理，大致实现思路是在系统调用到+load方法的时候，使用block来封装load方法里的逻辑，添加进一个全局的数组中，优先级设置是通过对全局数组排序事项的，最后在相应的时刻从该数组中取出block并执行。调用时机分别为以下三个：
* mian函数执行之前
* didFinishLaunch
* didFinishLaunch之后

例如：
```
+(void)load {
    imy_load_main(IMYLoadAtPreMain, 0, ^{
        [IMYBootingProtectionManager shareBootingProtectionManager];
    });
}
```
2、异常捕获
异常捕获，坑点太多，iOS系统允许我们去捕获mach级别和应用级别的异常。应用层的异常需要我们自己去注册一个异常捕获函数，但是一些三方sdk经常会注册该函数（例如友盟，bugly等），就会存在问题。前面注册的会被后面注册覆盖。你需要去写一些兼容代码。且自家的应用接入友盟，bugly这种三方统计的sdk。
通过hook NSSetUncaughtExceptionHandler这个函数（ [传送门](http://www.cocoachina.com/ios/20150701/12301.html)），清晰看到了app分别注册了三个handler，自己注册的，友盟注册的和bugly注册的。发生异常的时候，最终都被bugly的给拦截了。而进入到自己注册的异常处理函数。
Q:友盟是怎么统计到的？
A:这就很尴尬了。发现当bugly检测到异常，它会把该异常同时上报给友盟。
所以为了取巧，监控异常这部分通过初始化，由外部传入异常回调的函数从而达到监控异常的目的。
```
  imy_load_main(IMYLoadAtPreMain, 0, ^{
        // 注册安全模式
        [[IMY_AtomSDK shareInstance] regiterCrashProtectService];
        // 异常捕获回调   
        [[IMYAppReport shareInstance] addCrashReportLog:^NSString *_Nonnull(NSException *_Nonnull exception) {
            [[IMY_AtomSDK shareInstance] regiterSafeModeService];
            return nil;
        }];
    });

```

3、如何安全检测？
既然保证了app在启动之前都会进入安全模式。进入该模式的时候，首先会维护一个闪退计数器，并且开启一个定时任务，若在该时间段了，app没有检测到任何异常发生，则会将该计数器清0。若检测到了闪退，则将该计数器+1。逻辑见上图。

4、如何修复程序？
进入安全模式，获取闪退计数，在这里分别设置了三级保护措施，当闪退技术>=2的时候，则会进入三级修复，会删除沙盒里的接口数据缓存文件。当闪退技术>=3的时候，会删除接口数据缓存，同时也会删除JSpatch和RN的包。当闪退技术>=5的时候，则会清空整个沙盒。修复完成之后，则会将闪退计数置为0。而应用则会恢复到初始状态。逻辑见上图。

### 进程守护

#### 1、UI_Protect

闪退场景：
iOS 上不建议在非主线程进行UI操作，在非主线程进行UI操作有很大几率会导致程序崩溃，或者出现预期之外的效果。

实现思路：
通过Hook UIView类和CALayer类的一些方法来避免在非主线程执行UI操作。
```
UIView
1.setNeedsLayout
2.setNeedsDisplay
3.setNeedsUpdateConstraints
4.setNeedsDisplayInRect:

CALayer
1.setNeedsLayout
2.setNeedsDisplay
3.setNeedsDisplayInRect:
```

核心代码：
```
判断是否在主线程，不在则转移到主线程执行
 if (strcmp(dispatch_queue_get_label(DISPATCH_CURRENT_QUEUE_LABEL), dispatch_queue_get_label(dispatch_get_main_queue())) == 0) {
                                            // 调用原来的方法
                                        } else {
                                            dispatch_async(dispatch_get_main_queue(), ^{
                                                 // 调用原来的方法
                                            });
                                        }
```
#### 2、NSTimer_Protect

闪退场景：
NSTimer这个鬼使用的时候，需要非常小心，最常见的两个坑。
1.NSTimer会对target对象进行强引用，容易出现reatain circle。导致target无法释放，从而产生内存泄露。
2.在使用scheduledTimerWithTimeInterval:target:selector:userInfo:repeats:做一个重复的定时任务的时候， repeats参数设置为YES，在你不需要的时候，要你手动调用invalidate来销毁定时器。因为定时器会被放进Runloop中，如果target对象释放了之后，Runloop触发了定时器的定时任务，会因为找不到target对象而引起crash。

实现思路：
![Snip20170918_29.png](http://upload-images.jianshu.io/upload_images/1488967-2bf0a205872b5708.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主要解决两个问题：
1.NSTimer对Target的强引用
![Snip20170918_33.png](http://upload-images.jianshu.io/upload_images/1488967-4bdef0bcb66acd4a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.自动invalidate的NSTimer

![Snip20170918_34.png](http://upload-images.jianshu.io/upload_images/1488967-02cbd5ccfff260c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

核心代码：
```
1.hook
 static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        RSSwizzleClassMethod([NSTimer class], @selector(scheduledTimerWithTimeInterval:target:selector:userInfo:repeats:), RSSWReturnType(NSTimer *), RSSWArguments(NSTimeInterval ti, id aTarget, SEL aSelector, id userInfo, BOOL yesOrNo), RSSWReplacement({
                                 if (!_NSTimerCrashProtectEnable) {
                                     return RSSWCallOriginal(ti, aTarget, aSelector, userInfo, yesOrNo);
                                 }
                                 NSTimer *timer = nil;
                                 if (yesOrNo) {
                                     @autoreleasepool {
                                         IMY_TimerStubProxy *proxy = [[IMY_TimerStubProxy alloc] init];
                                         proxy.target = aTarget;
                                         proxy.aSelector = aSelector;
                                         timer.timerProxy = proxy;
                                         timer = RSSWCallOriginal(ti, proxy, @selector(fireProxyTimer:), userInfo, yesOrNo);
                                         proxy.sourceTimer = timer;
                                     }
                                     return timer;
                                 } else {
                                     return RSSWCallOriginal(ti, aTarget, aSelector, userInfo, yesOrNo);
                                 }
                                 return nil;
                             }));
        _NSTimerCrashProtectEnable = YES;
    });

2. fireProxyTimer：
- (void)fireProxyTimer:(id)userinfo
{
    id strongTarget = self.target;
    if (strongTarget && ([strongTarget respondsToSelector:self.aSelector])) {
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [strongTarget performSelector:self.aSelector
                           withObject:userinfo];
#pragma clang diagnostic pop

    } else {
        if (self.sourceTimer) {
            [self.sourceTimer invalidate];
        }
        NSString *reason = [NSString stringWithFormat:@"[AtomSDK]: Target is <%@> Method is <%@>, reason : an object dealloc not invalidate Timer.",
                                                      [self class], NSStringFromSelector(self.aSelector)];
        NSLog(@"%@", reason);
    }
}


```
#### 3、NSNull_Protect

闪退场景：
iOS中数组，字典等一些集合类，元素只能为OC对象，所以只能用NSNull来填空。在做数据解析的时候，经常遇到服务端返回的数据中带有null这种鬼，会被解析成NSNull这种对象。稍微不注意类型保护就会发生闪退。

实现思路：
hook NSNull类的forwardingTargetForSelector:，将一些无法识别的消息发送给一些可以识别的集合类。

核心代码：
```
 RSSwizzleInstanceMethod([NSNull class], @selector(forwardingTargetForSelector:), RSSWReturnType(id), RSSWArguments(SEL aSelector), RSSWReplacement({

                                    if (!_NSNullCrashProtectEnable) {
                                        return RSSWCallOriginal(aSelector);
                                    }
                                    static NSArray *sTmpOutput = nil;
                                    if (sTmpOutput == nil) {
                                        sTmpOutput = @[@"", @0, @[], @{}];
                                    }

                                    for (id tmpObj in sTmpOutput) {
                                        if ([tmpObj respondsToSelector:aSelector]) {
                                            return tmpObj;
                                        }
                                    }
                                    return RSSWCallOriginal(aSelector);
                                }),
                                RSSwizzleModeAlways, nil);
```

#### 4、NSNotification_Protect

闪退场景：
在iOS9之前，通过addObserver:selector:name:object:监听通知。NSNotificatinonCenter会对观察者进行强引用，所以需要手动的在观察者对象挂了的时候移除通知监听。如果未移除监听，在观察者被释放的之后仍然会接收到通知那么就会发生闪退。

实现思路：
主要是hook NSNotificationCenter的addObserver:selector:name:object:，凡是有添加过通知监听的对象，都会给该对象设置一个标识。在dealloc的hook中，判断需要释放的对象，是否有注册过通知监听，是则去移除通知。

#### 5、UnrecognizedSelector_Protect

闪退场景：
天天见。
![Snip20170925_22.png](http://upload-images.jianshu.io/upload_images/1488967-19552be78073a7dd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/900)

实现思路：
由于OC是一门动态类型的语言，它的方法调用其实是一种动态绑定的机制，也就是说方法调用是直到运行期才去确定的。
方法在调用时，系统会查看这个对象能否接收这个消息（查看这个类有没有这个方法，或者有没有实现这个方法。），牛逼的是，如果不能并且只在不能的情况下，就会调用下面这几个方法，给你“补救”的机会，你可以理解为几套防止程序crash的备选方案，苹果就是利用这几个方案进行消息转发，前一套方案实现后一套方法就不会执行。如果这几套方案你都没有做处理，那么程序就会报错crash。
完整的消息转发的流程如下：
![Snip20170925_23.png](http://upload-images.jianshu.io/upload_images/1488967-c0794df51a9ec109.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

主要是通过hook NSObject类的forwardingTargetForSelector：实例方法以及类方法。在该方法中将一些无法识别的方法，发送给一个stubProxy对象，且在运行时动态的给改stubProxy对象添加改SEL对应的方法实现(IMP)。

![Snip20170920_5.png](http://upload-images.jianshu.io/upload_images/1488967-b68da4e515176f62.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


核心代码：
```
    RSSwizzleInstanceMethod([NSObject class], @selector(forwardingTargetForSelector:), RSSWReturnType(id), RSSWArguments(SEL aSelector), RSSWReplacement({
                                    if (!_UnrecognizedSelectorCrashProtectEnable) {
                                        return RSSWCallOriginal(aSelector);
                                    }

                                    BOOL aBool = [self respondsToSelector:aSelector];
                                    // 获取消息签名
                                    NSMethodSignature *signatrue = [self methodSignatureForSelector:aSelector];

                                    if (aBool || signatrue) {
                                        return RSSWCallOriginal(aSelector);
                                    } else {
                                        IMY_UnrecognizedSelector_StubProxy *stub = [[IMY_UnrecognizedSelector_StubProxy alloc] init];
                                        [stub addFunc:aSelector];
                                        NSString *reason = [NSString stringWithFormat:@"[AtomSDK]: unrecognizedSelector <%@> to instance <%@>",
                                                                                      NSStringFromSelector(aSelector), [self class]];
                                        NSLog(@"%@", reason);
                                        return stub;
                                    }

                                }),
                                RSSwizzleModeAlways, nil);
```

#### 6、KVO_Protect

KVO 是 Objective-C 对观察者设计模式的一种实现。观察对象(例如A类)，当对象某个属性(例如A中的字符串name)发生更改时，监听对象会获得通知，并作出相应处理。且不需要给被观察的对象添加任何额外代码，就能使用的一种观察者模式。

闪退场景：
* 1.被观察者提前释放
```
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'An instance 0x600000018180 of class Student was deallocated while key value observers were still registered with it. Current observation info: <NSKeyValueObservationInfo 0x600000032340> (
<NSKeyValueObservance 0x600000054d60: Observer: 0x7fe684608b40, Key path: name, Options: <New: YES, Old: YES, Prior: NO> Context: 0x0, Property: 0x60000005cb00>
)'
```
* 2.被观察者是局部变量
* 3.重复移除观察者 
```
*** Terminating app due to uncaught exception 'NSRangeException', reason: 'Cannot remove an observer <TestViewController 0x7fdb13e0cf40> for the key path "view" from < TestViewController 0x7fdb13e0cf40> because it is not registered as an observer.'
```
* 4.观察者对象未实现observeValueForKeyPath:ofObject: change: context:方法
```
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: '< TestViewController: 0x7fc628f049e0>: An -observeValueForKeyPath:ofObject:change:context: message was received but not handled.
Key path: view
Observed object: < TestViewController: 0x7fc628f049e0>
Change: {
    kind = 1;
    new = "<UIView: 0x7fc628d18810; frame = (0 0; 414 736); layer = <CALayer: 0x60800002be00>>";
}
Context: 0x0'
```
ps:重复添加观察者并不会引起crash，但是会多次触发相应事件。


实现思路：
主要通过NSObject的分类，利用运行时给NSObject实例添加一个关联对象kvoProxy，该对象是用于管理当前对象的kvo关系(也就是观察者和被观察者的关系)，分别hook addObserver:forKeyPath:options:context:和removeObserver:forKeyPath:以及dealloc方法，在dealloc中移除真正的观察者对象。

![addObserver:forKeyPath:options:context:](http://upload-images.jianshu.io/upload_images/1488967-65735d525724bbcc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![removeObserver:forKeyPath](http://upload-images.jianshu.io/upload_images/1488967-72f9a4e0eb982742.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

核心代码：
```
    RSSwizzleInstanceMethod([NSObject class], @selector(addObserver:forKeyPath:options:context:), RSSWReturnType(void), RSSWArguments(NSObject * observer, NSString * keyPath, NSKeyValueObservingOptions options, void *context), RSSWReplacement({
                                // 忽略RAC
                                if (object_getClass(observer) == objc_getClass("RACKVOProxy")) {
                                    RSSWCallOriginal(observer, keyPath, options, context);
                                    return;
                                }
                                NSHashTable<NSObject *> *os = [self kvoProxy].kvoInfoMap[keyPath];
                                // 第一次的时候将KVOProxy添加为真正的观察者
                                if (os.count == 0) { // (包括了 observers == nil 和 count == 0)
                                    os = [[NSHashTable alloc] initWithOptions:(NSPointerFunctionsWeakMemory)capacity:0];
                                    [os addObject:observer];
                                    RSSWCallOriginal([self kvoProxy], keyPath, options, context);
                                    [self kvoProxy].kvoInfoMap[keyPath] = os;
                                    return;
                                }

                                if ([os containsObject:observer]) {
                                    // 找到同样的观察者 不重复添加
                                    NSString *reason = [NSString stringWithFormat:@"[AtomSDK]:KVO add Observer to many timers.Target is %@ method is %@ ",
                                                                                  [self class], NSStringFromSelector(@selector(addObserver:forKeyPath:options:context:))];
                                    NSLog(@"%@", reason);

                                } else {
                                    // 以后添加观察者直接往容器里面更新元素就行了
                                    [os addObject:observer];
                                }
                            }),
                            RSSwizzleModeAlways, nil);

    [RSSwizzle swizzleInstanceMethod:@selector(removeObserver:forKeyPath:)
                             inClass:[NSObject class]
                       newImpFactory:^id(RSSwizzleInfo *swizzleInfo) {
                           return ^void(__unsafe_unretained id self, NSObject *observer, NSString *keyPath) {


                               int (*originalIMP)(__unsafe_unretained id, SEL, NSObject *, NSString *);
                               originalIMP = (__typeof(originalIMP))[swizzleInfo getOriginalImplementation];
                               __imy_hook_orgin_function_removeObserver = originalIMP;


                               if (object_getClass(observer) == objc_getClass("RACKVOProxy")) {
                                   originalIMP(self, @selector(removeObserver:forKeyPath:), observer, keyPath);
                                   return;
                               }

                               NSHashTable<NSObject *> *os = [self kvoProxy].kvoInfoMap[keyPath];
                               if (os.count == 0) {
                                   // 未找到观察者
                                   NSString *reason = [NSString stringWithFormat:@"[AtomSDK]: KVO remove Observer to many times.target is %@ method is %@,",
                                                                                 [self class], NSStringFromSelector(@selector(removeObserver:forKeyPath:))];
                                   NSLog(@"[KVO]:%@", reason);

                                   return;
                               }
                               // 找到了观察者 移除
                               [os removeObject:observer];
                               // 为空时移除真正的观察者
                               if (os.count == 0) {
                                   //                                   NSLog(@"[KVO]:REMOVE【%@】-【%@】-【%@】",self,observer,keyPath);
                                   originalIMP(self, @selector(removeObserver:forKeyPath:), [self kvoProxy], keyPath);
                                   [[self kvoProxy].kvoInfoMap removeObjectForKey:keyPath];
                               }
                           };
                       }
                                mode:RSSwizzleModeAlways
                                 key:NULL];

```
#### 7、BadAccess_Protect

iOS中有空指针和野指针两种概念。空指针是没有存储任何内存地址的指针。如Student *s1 = NULL;和Student *s2 = nil;而野指针是指指向一个已删除的对象（”垃圾”内存既不可用内存）或未申请访问受限内存区域的指针。野指针是比较危险的。因为野指针指向的对象已经被释放了。通过野指针访问已经释放的对象crash其实不是必现的，因为dealloc执行后只是告诉系统，这片内存我不用了，而系统并没有就让这片内存不能访问。
BTW野指针的崩溃是比较随机的，你在测试的时候可能没发生crash，但是用户在使用的时候就可能发生crash了。

闪退场景：
野指针闪退常见有以下几种情况：
1.对象释放后内存没被改动过，原来的内存保存完好，可能不Crash或者出现逻辑错误（随机Crash）。
2.对象释放后内存没被改动过，但是它自己析构的时候已经删掉某些必要的东西，可能不Crash、Crash在访问依赖的对象比如类成员上、出现逻辑错误（随机Crash）。
3.对象释放后内存被改动过，写上了不可访问的数据，直接就出错了很可能Crash在objc_msgSend上面（必现Crash，常见）。
4.对象释放后内存被改动过，写上了可以访问的数据，可能不Crash、出现逻辑错误、间接访问到不可访问的数据（随机Crash）。
5.对象释放后内存被改动过，写上了可以访问的数据，但是再次访问的时候执行的代码把别的数据写坏了，遇到这种Crash只能哭了（随机Crash，难度大，概率低）！！
6.对象释放后再次release（几乎是必现Crash，但也有例外，很常见）。
![1467690258548317-1.gif](http://upload-images.jianshu.io/upload_images/1488967-f460ae871ef8b723.gif?imageMogr2/auto-orient/strip)


实现思路：

在app的当前进程中，如果我们额外向系统申请一块内存，用于存储那些已经被释放的对象。所以当一个对象被释放了时候，系统就会自动调用该对象的dealloc方法，在这个方法里面系统会释放该对象自身的一些属性，关联对象等等一些资源。然后，我们利用isa指针混淆技术，将该对象的类动态修改为一个僵尸类，且该僵尸对象可以处理任何方法。那么如果你向一个已经被释放的对象发送消息的时候，因为它存在于缓存池当中，那么也就不会产生野指针了。

首先我们需要知道dealloc的方法里面系统干了些什么事儿。
因为OC中内存管理采用的是引用计数器的方式，即每个对象的散列表中都会存在一个int类型的变量表示该对象的被引用的次数，当该对象的引用计数为0的时候，由系统自动调用该对象的dealloc方法释放该对象的内存。并且父类的dealloc的方法将在子类dealloc方法返回后自动调用。（PS:你在跟我讲故事么。）

通过查看Apple的runtime源码，可以看出ARC下的dealloc的实现原理。
```
- (void)dealloc {
    _objc_rootDealloc(self);
}

void _objc_rootDealloc(id obj)
{
    assert(obj);

    obj->rootDealloc();
}

inline void objc_object::rootDealloc()
{
    if (isTaggedPointer()) return;
    object_dispose((id)this);
}

id object_dispose(id obj) 
{
    if (UseGC) return _object_dispose(obj);
    else return (*_dealloc)(obj); 
}

static id _object_dispose(id anObject) 
{
    if (anObject==nil) return nil;

    objc_destructInstance(anObject);
    
#if SUPPORT_GC
    if (UseGC) {
        auto_zone_retain(gc_zone, anObject); // gc free expects rc==1
    } else 
#endif
    {
        // only clobber isa for non-gc
        anObject->initIsa(_objc_getFreedObjectClass ()); 
    }
    free(anObject);
    return nil;
}
```
这里能够清晰地看到在执行dealloc的时候，会先执行_objc_rootDealloc()函数。然后执行_object_dispose()，且在这个函数里干了三件事情：
* 1.调用objc_destructInstance()释放对象的所有实例变量和关联对象(该方法并未回收对象本身内存).
```
void *objc_destructInstance(id obj) 
{
    if (obj) {
        // Read all of the flags at once for performance.
        bool cxx = obj->hasCxxDtor();
        bool assoc = !UseGC && obj->hasAssociatedObjects();
        bool dealloc = !UseGC;

        // This order is important.
        if (cxx) object_cxxDestruct(obj);
        if (assoc) _object_remove_assocations(obj);
        if (dealloc) obj->clearDeallocating();
    }
    return obj;
}
```
* 2.isa-swizzling将该对象的类置为一个空的类对象.
* 3.调用free()回收该对象的内存.

所以我们需要在对象真正被释放的时机搞事情，有三个选择：
* 1.运行时hook NSObject的dealloc方法。
* 2.runtime object_dispose 。
* 3.利用fishhook c的free()。
  这三个调用栈顺序依次dealloc->object_dispose->free。所以还是选择hook最上层dealloc方法。主要针对OC对象的野指针。
  流程图如下：
  ![Snip20170929_35.png](http://upload-images.jianshu.io/upload_images/1488967-4d3a9e5d5c875e27.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/800/h/800)


核心代码：
```
- (void)IMY_BadAccess_dealloc {
    NSObject *obj = (NSObject *)self;
    objc_destructInstance(self);
    object_setClass(self, [_IMYZombieObj class]);
    obj.oriClass = NSStringFromClass([self class]);
    dispatch_async(zombieOperationQueue, ^{
        if (global_unfree_list->unfree_count > MAX_UNFREE_POINTER * 0.9 || global_unfree_list->unfree_size > MAX_UNFREE_MEM) {
            freeMemInListSync(global_unfree_list, FREE_POINTER_NUM);
        }
        addUnFreeMemToListSync(global_unfree_list, (__bridge void *)(self));
    });
}
```
