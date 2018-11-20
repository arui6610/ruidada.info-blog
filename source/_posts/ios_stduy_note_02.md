---
layout: post
title: "学习笔记02：OC的消息机制"
date: 2015-05-11 14:36
categories: 
- 技术
- [技术, OC]
tags: 
- 技术
- OC
- 笔记
---
举个栗子！！！
Person.h
```objective-c
#import <Foundation/Foundation.h>
@interface Person : NSObject
- (void)eat;
@end
```
Person.m
``` objective-c
#import "Person.h"
@implementation Person
- (void)eat {
    NSLog(@"eat...");
}
@end
```
<!-- more -->
Main.m
```objective-c
#import <Foundation/Foundation.h>
#import "Person.h"
int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Person *p = [[Person alloc] init];
    }
    return 0;
}
```

#### 1.如何调用p对象的eat方法呢？除了[p eat]这种方式，还有没有其他方式能够调用eat方法？如果我在.h文件中没有eat方法的声明，又如何调用？

* 1>直接调用

  ```objective-c
  [p eat];
  ```


* 2>通过performSelector

  ```objective-c
  if ([p respondsToSelector:@selector(eat)]) { // 判断方法是否存在，防止方法不存在导致闪退
  	[p performSelector:@selector(eat)];
  }
  ```


* 3>利用NSInvovation

  ```objective-c
  // 获取方法签名
  NSMethodSignature *sign = [[p class] instanceMethodSignatureForSelector:selector];
  // 创建invocation对象
  NSInvocation *eatIvo = [NSInvocation invocationWithMethodSignature:sign];
  eatIvo.target = p;
  eatIvo.selector = selector;
  [eatIvo invoke];
  ```

* 4>运行时消息函数

  ```objective-c
   objc_msgSend(p,@selector(eat));
  ```
**PS:**
在Xcode7中直接调用objc_msgSend函数会报错。主要是因为编译器会检查你objc_msgSend函数调用。解决办法有2种：

  * 1.强转  

       ```objective-c
       Class class = [p class];
       ((void * ()(void, SEL))(void*)objc_msgSend)(&class, @selector(eat));
       ```

  * 2.Build Settings->Enable Strict Checking of objec_msgSend Calls 设置为YES

方法调用的几种方式。那么一开始的那个疑问，如果-eat方法只有实现，在.h文件中而没有声明，这样外部也就无法调用到-eat方法了。当然还是可以利用上述2，3，4种方式来调用。又在想，如果做绝一点，连Person类都不让你知道呢？我又该怎么创建Person类的实例？怎么调用p对象的那个-eat方法呢？这时候就需要用到运行时。获取到类名还有类的所有方法名。利用反射将字符串转变为类，方法等等。同样可以利用2,3,4来调用方法。那是不是可以认为在OC中没有严格意义上的私有方法呢。

#### 2.OC中的SEL是什么东西？方法和函数有什么关系？动态绑定是什么意思？为什么OC中的方法调用也叫发消息？那么消息是怎么传递的呢？消息转发又是什么鬼？还有消息机制的完整流程是几个意思？
一大串的问题。对于消息这东西，思绪是乱的。静下心来一点点的梳理一遍，也是受益很多，对于OC也有了更深的理解。
* 
  1.SEL是什么？SEL也叫方法选择器，它的定义如下：
  ```c
  typedef struct objc_selector *SEL;
  ```
  其实就是一个特殊的结构体类型的指针，而该结构体就是对每个方法的方法名，参数序列等等一些信息的描述。

  关于SEL其实就是一个根据方法名哈希处理后的一个用来标识某个方法的一个唯一标识而已。一个SEL就代表着一个方法。通过这个SEL就能找到对应的方法实现。使用哈希处理后的方法名而不用方法名去做标识，这样提高了查找方法实现的效率。
  ps：

  * @selector()只是个编译器指令。在编译期会自动将这部分转换为对应的代码。
  * 在某一个类或该类的继承体系中，如果出现两个方法名相同的方法，即使参数不同。他们生成的SEL也是一样的。所以OC中某一个类或该类的继承体系中最好不要出现两个相同方法名的方法。

* 2.动态绑定
  OC既然是动态语言，重要的一个体现就是动态绑定。OC是在运行时确定对象所属的真实类型。确定类型之后，才会将对应的属性和响应的消息绑定到该对象上。

* 3.方法
  我们知道类的结构中有一个MethodLists表。里面存储的都是一个个方法。该结构体中主要就SEL（方法选择）和IMP（方法实现）。简单理解该结构体就相当于是在SEL和IMP之间做了一个映射。方便我们通过SEL找到对应的IMP。

  内存结构如下：

  ```c
   typedef struct objc_method *Method;
   struct objc_method {
      SEL method_name                 OBJC2_UNAVAILABLE;  // 方法名
      char *method_types                  OBJC2_UNAVAILABLE;
      IMP method_imp                      OBJC2_UNAVAILABLE;  // 方法实现
  }
  ```


* 4.发消息
  [p eat];通常我们说对象p调用了eat方法，也可以说向p对象发送了一条eat的消息。这两种说法。我觉得后面那种方式更加形象也更加准确。
  那这句话的背后系统又干了些什么？是时候推开门进去看一看了。
  在编译时期，编译器会将大部分的方法转换为函数，如objc_msgSend(),obj_msgSendSuper()等等。而在运行时期，消息才会被绑定到方法上。
  例如：
      [p eat];
      在编译时则会转换为：objc_msgSend(p,@selector(eat));这样的函数。
  objc_msgSend()是什么？
  ```
  objc_msgSend(receiver, selector, arg1, arg2, ...)
  receiver：消息接收者
  selector：方法选择器
  arg：其他参数
  ```




* 5.消息传递
  画个图吧。画好了。


![消息传递过程.png](http://upload-images.jianshu.io/upload_images/1488967-adec86257f242c49.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


因为OC的动态特性。所以它会运行时确定该对象或该类所属的真实类型，也就是根isa指针去找它的真实类型。确定类型后，首先它会去cache表里面根据SEL去找IMP(方法实现函数)。没有的话，她就会去MethodLists里面找。找到了就去执行对应的IMP，并且她会将这个方法扔到Cache里面。如果在MethodLists里面也没找着。她回去根据superclass区她的父类按照这个顺序去查找方法实现。直到根类为止。从这里也就能看出来OC的方法调用其实走的是一个缓存机制。提高了方法调用的效率。

* 消息转发
  刚才说到了消息传递。有个疑问。如果在在Person.m中干掉-eat方法实现呢？那么整套流程走了下来。都没有找到那个方法会出现什么情况？答案当然是unrecognized selector sent to instance 0x2b2b2b2b..类似的字样，并且闪退。
  所以，系统在经过消息传递过程后，还没有找到这个方法，那么它又偷偷摸摸的干了些什么呢？给不给机会挽回一下呢？
  点开NSObject类的头文件。

  ![NSObject.png](http://upload-images.jianshu.io/upload_images/1488967-1ada22c46563f0bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

发现了四个方法。这四个方法也就是用来实现消息转发的。也是用来给你没有实现那个方法的过错进行补救的。当然OC是人性化的。她一共给了你三次机会。消息转发的具体步骤如下：
1.方案一。如果是实例方法，会调用+ (BOOL)resolveInstanceMethod:(SEL)sel；方法。如果是类方法，则调用+ (BOOL)resolveClassMethod:(SEL)sel；
这两个方法的作用也就是让你在程序运行时动态的为一个selector提供实现。
栗如上面的那个疑问。我在Person.m中干掉了-eat的实现。则可以通过重载这个方法动态添加-eat的方法实现。

```objective-c
@implementation Person
void eatIMP(id self ,SEL _cmd) {
    NSLog(@"eat...");
}
+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(eat)) {
        class_addMethod([self class], sel, (IMP)eatIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
@end
```

 这里用到了class_addMethod()函数；
```          objective-c
BOOL class_addMethod(Class cls, SEL name, IMP imp, const char *types)
cls：添加方法到指定的类。
name：方法选择器
imp：方法实现
types：请参考Objective-C Runtime Programming Guide-类型编码
```
([ Objective-C Runtime Programming Guide-类型编码](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html ))

这种方式让我们有机会在运行时通过resolveInstanceMethod或resolveClassMethod中动态的将某个方法实现添加到类中。那问题就来了。就一个方法实现而已。还要弄这么多代码。有毛用啊？
这个动态方法解析主要是为了解决@dynamic属性而存在的。

那么我又要举手了。@dynamic又是什么鬼？
@dynamic 其实就是要是关键字。是相对于 @synthesize 的，它们用样用于修饰 @property，用于生成对应的的 getter 和 setter 方法。但是 @ dynamic 表示这个成员变量的 getter 和 setter 方法并不是直接由编译器生成，而是手工生成或者运行时生成。


2.方案二
如果方案一你自己没有处理的话。就会来到- (id)forwardingTargetForSelector:(SEL)selector；这个方法。这个方法主要是用来将这个消息发送给一个可以处理的对象的。栗如：我收到了一个bug，我处理不了，我基友说他可以。那我通过这个方法我就可以把这个bug转发给我基友了。
栗如：
```objective-c
Person.h

#import <Foundation/Foundation.h>
#import "Dog.h"
@interface Person : NSObject
@property (nonatomic, strong) Dog *dog;
- (void)eat;
@end

Person.m

#import "Person.h"
@implementation Person

- (void)eat {
    NSLog(@"eat...");
}

- (id)forwardingTargetForSelector:(SEL)aSelector {
    if (aSelector == @selector(bark)) {
        return self.dog;
    }
    return [super forwardingTargetForSelector:aSelector];
}
@end

```

```objective-c
Dog.h

#import <Foundation/Foundation.h>
@interface Dog : NSObject
- (void)bark;
@end


Dog.m

#import "Dog.h"
@implementation Dog
- (void)bark {
    NSLog(@"wangwang...");
}
@end
```

```
main.m

#import <Foundation/Foundation.h>
#import "Person.h"

int main(int argc, const char * argv[]) {
    @autoreleasepool {
        Dog *dog = [[Dog alloc] init];
        Person *oldWang = [[Person alloc] init];
        oldWang.dog = dog;
        
        [oldWang performSelector:@selector(bark)];
    }
    return 0;
}
```
人有一只狗。狗有一个叫的方法。在main.m中我分别创建了一个Dog实例，还有个Person实例。并且将dog赋值给了oldWang的dog属性是不是。
那么现在oldWang有了一条dog。[oldWang performSelector:@selector(bark)];我们让oldWang像狗狗一样的叫。很明显不可能的是吧。所以我们在Person中重载了forwardingTargetForSelector方法。在这里面将bark这条消息发送给了dog属性。

PS:
这种方式看起来蛮有意思的。混搭有没有。人可以飞。猪可以上树。但是有一个很大的局限性。具体执行这些行为的都不是对象本身。而是该对象持有的其他对象或者其他对象。感觉有点狐假虎威。

使用场景：这种方式只适用于我们只想将消息转发到另一个能处理该消息的对象上，而并不能处理消息的参数和返回值。

方案三：
好吧。再假设你第一步和第二步都没有做处理。事不过三。那就给你最后一次机会。- (void)forwardInvocation:(NSInvocation *)anInvocation。

这一次运行时系统会将消息打包成一个NSInvocation对象。启动完整的消息转发机制。

那么forwardInvocation：干了些什么呢？
* 1.定位可以响应封装在anInvocation中的消息的对象。这个对象不需要能处理所有未知消息。
* 2.使用anInvocation作为参数，将消息发送到选中的对象。anInvocation将会保留调用结果，运行时系统会提取这一结果并将其发送到消息的原始发送者。

还是用代码说话。瞎逼逼谁不会啊。
还是刚才的代码。只不过改了下Person的实现文件。oldWang有一个dog。让oldWang像狗一样的叫。

```objective-c
#import "Person.h"

@implementation Person

- (void)eat {
    NSLog(@"eat...");
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    NSMethodSignature *sign = [super methodSignatureForSelector:aSelector];
    if (!sign) {
        sign = [[self.dog class] instanceMethodSignatureForSelector:@selector(bark)];
    }
    return sign;
}
- (void)forwardInvocation:(NSInvocation *)anInvocation {
    if (anInvocation) {
        [anInvocation invokeWithTarget:self.dog];
    }
}
@end
```

在这里，我们重载了methodSignatureForSelector方法，这里需要返回一个非nil的NSMethodSignature对象。然后在forwardInvocation里进行转发消息。所以你可以在methodSignatureForSelector方法里对所有未识别的消息进行签名。感觉就像是个加工厂。消息想要进入forwardInvocation转发前必须经过这里包装下。接下来运行时系统会将这里NSMethodSignature对象自动打包成一个个NSInvocation对象。进入forwardInvocation，将消息发送出去。

其实这里，一开始我有过一个疑问，既然OC不支持多继承。那么通过消息转发这种形式，不是很像是多继承吗？后来发现，它和多继承最大区别是在于，多继承是在于集中。而这种方式却更像是在分散。将不同的功能分散到不同的对象身上。

到了这里OC中的消息转发也差不多了。那么我又有疑问了。知道了这么多，有毛用？不知道我也能干活。
通过查阅资料，Demo。总结了消息转发的几个用途。
* 创建一个对象负责把消息转发给一个由其它对象组成的响应链，代理对象会在这个有其它对象组成的集合里寻找能够处理该消息的对象； 
* 把一个对象包在一个logger对象里，用来拦截或者纪录一些有趣的消息调用；
* 比如声明类的属性为dynamic，使用自定义的方法来截取和取代由系统自动生成的getter和setter方法。



![异常.jpg](http://upload-images.jianshu.io/upload_images/1488967-0707de44c067caeb.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
