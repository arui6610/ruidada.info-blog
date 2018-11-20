---
layout: post
title: "日常开发01：一行代码搞定全屏滑动返回手势"
date: 2016-01-10 13:36:30
categories: 
- 技术
- [技术, iOS]
tags: 
- 技术
- iOS
- 日常
---

开发过程中我们经常会遇到这种需求，给某个页面添加全屏的滑动返回。当然iOS7之后，系统有提供一个边缘滑动返回的手势。很明显无法完成需求。产品要的是全屏。

* 思路1：
   既然UINavigationController有提供interactivePopGestureRecognizer手势。 UIGestureRecognizer采用的是target-action。这样我们可以找到手势的target和action。然后新建一个自己的手势实例，替换为系统手势的action实现就是了。

   <!-- more -->

   ```
       id target = self.interactivePopGestureRecognizer.delegate

           UIPanGestureRecognizer *pan = [[UIPanGestureRecognizer alloc] initWithTarget:target action:@selector(handleNavigationTransition:)];

        pan.delegate = self;

           [self.view addGestureRecognizer:pan];

           self.interactivePopGestureRecognizer.enabled = NO;

   ```

   ​

* 思路2：
  能不能不写那么多的代码？在研究interactivePopGestureRecognizer手势的过程中。通过运行时层层打印。
  * _recognizer：
    ![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1488967-5c04f8190606dbed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * _recognizer._settings：
      ![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1488967-2a283ed9c0d7385b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  * _recognizer._settings._edgeSettings：
    ![Paste_Image.png](http://upload-images.jianshu.io/upload_images/1488967-b451632afa82d5b1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

猜测_edgeSettings属性就是我们要找的设置返回手势响应范围的。
```        
[self.interactivePopGestureRecognizer setValue:@([UIScreen mainScreen].bounds.size.width) forKeyPath:@"_recognizer._settings._edgeSettings._edgeRegionSize"];  
```
