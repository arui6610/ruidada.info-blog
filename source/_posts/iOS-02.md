---
layout: post
title: "Main函数之前的那些事儿01：dylb"
date: 2016-05-10 08:10
categories: 
- 技术
- [技术, iOS]
tags: 
- 技术
- iOS
---
>前言：
>甩几个问题出来，答案从问题开始。
>1.类的+load方法系统是怎么调用的？
>2.一个类+load方法系统为什么只会调用一次？为什么不能在+load方法中调用super？
>3.为什么fishhook库只能hook应用挂载的外部动态库里的函数？
>
>然后这些问题就引出了dylb这个鬼，然后就一脸懵逼了。iOS应用中可执行文件是什么？有那些是可执行文件？动态库是在程序启动后由dylb链接，那么静态库又是什么时候链接的？dylb是通过镜像的形式加载二进制文件的，镜像是什么鬼？运行时环境也是在mian之前初始化的，dylb和运行时环境有没有什么不可描述的关系？

---

**编译-链接-运行**
源文件-->编译-->目标文件-->（静态链接）链接静态库文件-->生成可执行文件
app启动-->加载可执行文件-->（动态链接）链接动态库文件-->初始化运行时环境-->main()

<!-- more -->

**Helloworld**
![Snip20170505_2.png](http://upload-images.jianshu.io/upload_images/1488967-838b458845226301.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![Snip20170505_4.png](http://upload-images.jianshu.io/upload_images/1488967-1447ca6d90136428.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


1.分别创建了Foo.h,Foo.m,helloworld.m三个文件
2.编译Foo.m和helloworld.m文件，生成Foo.o和helloworld.o两个目标文件
3.链接Foundation库，如果不指定默认生成a.out的可执行文件
4.执行可执行文件

**Mach-O文件类型**
  *  1.可执行文件（mh_execute）(.app)
     简单理解为:静态库中所有的.o文件+app编译产生的所有.o文件的集合
  *  2.动态库（mh_dylib）（.dylib）
  *  3.包（mh_bundle）(.bundle)
      简单理解，就是资源文件包 
  *  4.静态库（staticlib）(.a/.framework)
     .a:一堆.o文件的集合
      .framework：一堆.o文件的集合+资源文件+头文件
  *  5.可重定位的对象文件（mh_object）(.o)
     编译后，每一个.m/.c/.mm等文件都会生成为一种.o格式的目标文件。

**Mach-O文件结构**

![Mach-O文件结构.png](http://upload-images.jianshu.io/upload_images/1488967-459b57d7460696ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/680)

* 1.标题:指定文件的目标体系结构，如PPC，PPC64，IA-32或x86-64。
* 2.加载命令：在虚拟内存中指定文件的逻辑结构和文件的布局。
* 3.原始段数据：包含在加载命令中定义的段的原始数据。

**静态库（.a /.framework）**
链接时期由静态链接器将静态库中的.o文件完整复制到可执行文件中。

***PS:CocoaPods 使用编译静态库 .a 方法将代码集成到项目中。在 Pods 项目中的每个 target 都对应这一个 Pod 的静态库。不过在编译过程中并不会真的产出 .a 文件。如果需要 .a 文件的话，可以参考这里，或者使用CocoasPods-Packager这个插件。（所以每次pod install后，特么编译项目都贼慢，为毛，因为编译器需要编译项目本身代码+各自pod库中的代码，最后将pod库中的代码生成.a文件。然后再编译就会快很多啊，因为编译时期就没有编译pod库了，而是直接连接一个个pod的.a库）***

**动态库(.dylib/.framework)**
由dylb动态加载进内存。且多个app共享一份动态库，动态库基本上都是由系统提供。
动态库的优点是，不需要拷贝到目标程序中，不会影响目标程序的体积，而且同一份库可以被多个程序使用（因为这个原因，动态库也被称作共享库）。同时，编译时才载入的特性，也可以让我们随时对库进行替换，而不需要重新编译代码。动态库带来的问题主要是，动态载入会带来一部分性能损失，使用动态库也会使得程序依赖于外部环境。如果环境缺少动态库或者库的版本不正确，就会导致程序无法运行。

***PS:iOS 平台不支持使用动态库，开发者可以使用的 Framework 只有苹果自家的 UIKit.Framework，Foundation.Framework 等。这种限制可能是出于安全的考虑。因为 iOS 应用都是运行在沙盒当中，不同的程序之间不能共享代码，动态下载代码又是被苹果明令禁止的。***

**静态链接(static linking)**
静态连接器就是将项目自身编译好的.o文件集合和一些静态库的.o文件集合进行整合，然后提取出每个.o文件中的用到的一些外部符号引用，并且对这些符号进行重定位，最终生成一个完整的可执行文件。

***PS:生成并非是真正的可执行文件，因为此时并不包含一些动态库中的代码，保存的只是一些对外部库中的符号引用。***

**动态连接(dylb)**

![forumImage20161207173929050.png](http://upload-images.jianshu.io/upload_images/1488967-add412737218082e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

内核初始化环境之后，初始化的dylb环境，由dylb加载一些系统库以及动态库到内存中之后，然后初始化运行时环境，接下来由运行时环境接管，通过运行时环境调用所有class的+load方法。_dylb_start之前的部分属于xnu内核的东西，看不太懂。

*  动态链接库加载流程
  *  load dylibs image 读取库镜像文件
      * 1.分析所依赖的动态库
      * 2.找到动态库的mach-o文件
      * 3.打开文件
      * 4.验证文件
      * 5.在系统核心注册文件签名
      * 6.对动态库的每一个segment调用mmap()
  *  Rebase image
  *  Bind image
  *  Objc setup
      * 1.注册Objc类 (class registration)
      * 2.把category的定义插入方法列表 (category registration)
      * 3.保证每一个selector唯一 (selctor uniquing）

  *  initializers
    * 1.Objc的+load()函数
    * 2.C++的构造函数属性函数 形如**attribute**((constructor)) void DoSomeInitializationWork()
    * 3.非基本类型的C++静态全局变量的创建(通常是类或结构体)(non-trivial initializer) 比如一个全局静态结构体的构建，如果在构造函数中有繁重的工作，那么会拖慢启动速度

在load dylibs image中可以看到dylb去读取镜像文件，加载进内存，并且对Mach-O文件进行解析。而镜像也就是指Mach-O文件。且不同类型的Mach-O文件由不同的ImageLoader子类实例进行加载。

![forumImage20161207151029417.png](http://upload-images.jianshu.io/upload_images/1488967-ba7a0f193fdb7a99.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Rebase image和Bind image是对镜像文件进行一些优化计算之类的操作。

* 运行时处理+load加载

![forumImage20161207173728453.png](http://upload-images.jianshu.io/upload_images/1488967-64d6772684462a77.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 运行时初始化的时候，会向dylb注册镜像加载的回调。也就是每当有dylb有新的镜像加载进内场的时候，新的镜像都会被map到runtime中，并且由runtime处理+load的加载。

利用https://github.com/RetVal 封装好的 debug 版最新源码可以进行断点调试，就可以看到系统在调用Test类的+load()方法之前的调用栈。打印如下：
```
    frame #0: 0x0000000100000f37 debug-objc`+[Test load](self=Test, _cmd="load") + 23 at Test.m:13
    frame #1: 0x00000001000b84e6 libobjc.A.dylib`call_class_loads() + 198 at objc-loadmethod.mm:209
    frame #2: 0x00000001000a2a4d libobjc.A.dylib`::call_load_methods() + 77 at objc-loadmethod.mm:361
    frame #3: 0x00000001000a28ba libobjc.A.dylib`::load_images(path="/Users/xurui/Library/Developer/Xcode/DerivedData/objc-eaekfqpmgbdaycbpoisopskbjiqq/Build/Products/Debug/debug-objc", mh=0x0000000100000000) + 106 at objc-runtime-new.mm:2051
  * frame #4: 0x0000000100007072 dyld`dyld::notifySingle(dyld_image_states, ImageLoader const*, ImageLoader::InitializerTimingList*) + 439
    frame #5: 0x00000001000161dc dyld`ImageLoader::recursiveInitialization(ImageLoader::LinkContext const&, unsigned int, char const*, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 320
    frame #6: 0x0000000100015268 dyld`ImageLoader::processInitializers(ImageLoader::LinkContext const&, unsigned int, ImageLoader::InitializerTimingList&, ImageLoader::UninitedUpwards&) + 138
    frame #7: 0x00000001000152fd dyld`ImageLoader::runInitializers(ImageLoader::LinkContext const&, ImageLoader::InitializerTimingList&) + 75
    frame #8: 0x000000010000747a dyld`dyld::initializeMainExecutable() + 195
    frame #9: 0x000000010000b7e0 dyld`dyld::_main(macho_header const*, unsigned long, int, char const**, char const**, char const**, unsigned long*) + 3928
    frame #10: 0x0000000100006249 dyld`dyldbootstrap::start(macho_header const*, int, char const**, long, macho_header const*, unsigned long*) + 470
    frame #11: 0x0000000100006036 dyld`_dyld_start + 54

```
4-11步骤主要执行的是_dyld_start的一些初始化操作。
主要看0-3步骤。
* load_images
  源码解读：
```
void load_images(const char *path __unused, const struct mach_header *mh)
{
    // 没有查询到传入 Class 中的 load 方法，视为锁定状态
    // 则无需给其加载权限，直接返回
    if (!hasLoadMethods((const headerType *)mh)) return;
    // 定义可递归锁对象
    // 由于 load_images 方法由 dyld 进行回调，所以数据需上锁才能保证线程安全
    // 为了防止多次加锁造成的死锁情况，使用可递归锁解决
    recursive_mutex_locker_t lock(loadMethodLock);

    // 收集所有的 +load 方法
    {
        // 对 Darwin 提供的线程写锁的封装类
        rwlock_writer_t lock2(runtimeLock);
        // 提前准备好满足 +load 方法调用条件的 Class
        prepare_load_methods((const headerType *)mh);
    }

    // 调用 +load 方法 (without runtimeLock - re-entrant)
    call_load_methods();
}
```
* prepare_load_methods
  源码解读：
```
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;

    runtimeLock.assertWriting();
    // 收集 Class 中的 +load 方法
    // 获取所有的类的列表
    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        // 通过 remapClass 获取类指针
        // schedul_class_load 递归到父类逐层载入
        schedule_class_load(remapClass(classlist[i]));
    }
    // 收集 Category 中的 +load 方法
    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        // 通过 remapClass 获取 Category 对象存有的 Class 对象
        Class cls = remapClass(cat->cls);
        if (!cls) continue;
        // 对类进行第一次初始化，主要用来分配可读写数据空间并返回真正的类结构
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        // 将需要执行 load 的 Category 添加到一个全局列表中
        add_category_to_loadable_list(cat);
    }
}
```
* call_load_methods() 
  源码解读：
```
void call_load_methods(void)
{
    // 是否已经录入
    static bool loading = NO;
    // 是否有关联的 Category
    bool more_categories;

    loadMethodLock.assertLocked();

    // 由于 loading 是全局静态布尔量，如果已经录入方法则直接退出
    if (loading) return;
    loading = YES;
    // 声明一个 autoreleasePool 对象
    // 使用 push 操作其目的是为了创建一个新的 autoreleasePool 对象
    void *pool = objc_autoreleasePoolPush();

    do {
        // 重复调用 load 方法，直到没有
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 调用 Category 中的 load 方法
        more_categories = call_category_loads();

        // 继续调用，直到所有 Class 全部完成
    } while (loadable_classes_used > 0  ||  more_categories);
    // 将创建的 autoreleasePool 对象释放
    objc_autoreleasePoolPop(pool);
    // 更改全局标记，表示已经录入
    loading = NO;
}
```
* call_class_loads() 
  源码解读：
```
static void call_class_loads(void)
{
    // 声明下标偏移
    int i;
    
    // 分离加载的 Class 列表
    struct loadable_class *classes = loadable_classes;
    // 调用标记
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    
    // 调用列表中的 Class 类的 load 方法
    for (i = 0; i < used; i++) {
        // 获取 Class 指针
        Class cls = classes[i].cls;
        // 获取方法对象
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        // 方法调用
        (*load_method)(cls, SEL_load);
    }
    
    // 释放 Class 列表
    if (classes) free(classes);
}
```
+load方法的调用过程，在runtime接收到有dylb有新的镜像加载的回调之后，通过laodImage()将所有的class加载进内存中，并且按照继承结构，递归获取所有class的+load方法，将该class的+load方法添加进一个全局的loadable_classes的结构体中存放，将该class的分类中的+load方法添加一个全局的loadable_categories的结构体中存放。最后通过call_load_methods()函数查找class在内存中对应的结构体中的+load方法，依次执行+load方法的过程。


参考
https://www.mikeash.com/pyblog/friday-qa-2012-11-09-dyld-dynamic-linking-on-os-x.html
http://blog.sunnyxx.com/2014/08/30/objc-pre-main/
https://techblog.toutiao.com/2017/01/17/iosspeed/
