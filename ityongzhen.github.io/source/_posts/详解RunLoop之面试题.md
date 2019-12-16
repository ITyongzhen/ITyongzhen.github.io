---
layout: post
title: 详解RunLoop之面试题
date: 2017-05-30 17:32:24.000000000 +09:00
categories: 
- iOS
---

本文首发于[个人博客](https://ityongzhen.github.io/%E8%AF%A6%E8%A7%A3RunLoop%E4%B8%8E%E5%A4%9A%E7%BA%BF%E7%A8%8B.html)

## 回顾[详解RunLoop之源码分析](https://ityongzhen.github.io/%E8%AF%A6%E8%A7%A3RunLoop%E4%B9%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.html)中提出的问题

+ 什么是Runloop
+ ios程序中 main函数为什么不会退出
+ runloop和线程的关系？
+ 程序中添加每3秒响应一次的NSTimer，当拖动tableview时timer可能无法响应要怎么解决？
+ runloop内部实现逻辑？
+ runloop 是怎么响应用户操作的， 具体流程是什么样的？
+ 说说runLoop的几种状态
+ runloop的mode作用是什么？
+ 如何实现一个常驻线程

## 什么是Runloop

+ Runloop 还是比较顾名思义的一个东西，说白了就是一种循环，只不过它这种循环比较高级。一般的 while 循环会导致 CPU 进入忙等待状态，而 Runloop 则是一种“闲”等待，这部分可以类比 Linux 下的 epoll。当没有事件时，Runloop 会进入休眠状态，有事件发生时， Runloop 会去找对应的 Handler 处理事件。Runloop 可以让线程在需要做事的时候忙起来，不需要的话就让线程休眠
+ RunLoop是通过通过内部维护的时间循环来对事件/消息进行管理的一个对象
	+ 没有消息需要处理时，休眠避免掉资源占用
		+ 用户态 -> 内核态
	+ 有消息时候，立刻被唤醒 
		+ 内核态 -> 用户态 
![image.png](https://upload-images.jianshu.io/upload_images/3373351-a836912b87aa725b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## ios程序中 main函数为什么不会退出
 + main函数内部调用 UIApplicationMain 这个方法会启动一个RunLoop，有事做就做事，没事做就等待，保持不会退出
 
## runloop和线程的关系？
	+ 每条线程都有唯一的一个与之对应的RunLoop对象
	+ RunLoop保存在一个全局的Dictionary里，线程作为key，RunLoop作为value
	+ 线程刚创建时并没有RunLoop对象，RunLoop会在第一次获取它时创建
	+ RunLoop会在线程结束时销毁


## runloop内部实现逻辑？
就是下图所示的，用自己话总结出来就好
![image.png](https://upload-images.jianshu.io/upload_images/3373351-2cc435e0dbda8ba0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## 程序中添加每3秒响应一次的NSTimer，当拖动tableview时timer可能无法响应要怎么解决？
 + 常见的2种Mode

 + kCFRunLoopDefaultMode（NSDefaultRunLoopMode）：App的默认Mode，通常主线程是在这个Mode下运行

 + UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响
 + common模式下(一个占位用的Mode，不是一种真正的Mode) 可以兼容以上两种模式

## runloop 是怎么响应用户操作的， 具体流程是什么样的？
 + source1 捕捉用户触摸事件
 + source0去处理触摸时间

## 说说runLoop的几种状态
 +  kCFRunLoopEntry = (1UL << 0),           // 即将进入Loop
 +  kCFRunLoopBeforeTimers = (1UL << 1),    //即将处理Timer
 +  kCFRunLoopBeforeSources = (1UL << 2),   //即将处理Source
 +  kCFRunLoopBeforeWaiting = (1UL << 5),   //即将进入休眠
 +  kCFRunLoopAfterWaiting = (1UL << 6),    //刚从休眠中唤醒
 +  kCFRunLoopExit = (1UL << 7),            //即将退出Loop
 +  kCFRunLoopAllActivities = 0x0FFFFFFFU   //所有状态改变
 
## runloop的mode作用是什么？
+ 简单来说就是不同模式隔离开来，保证同一种摸下下运行，source0,source1,timer,observer 的运行更流畅

## 如何实现一个常驻线程
简单来说就是

+ 创建RunLoop 
+ 像RunLoop中添加port、source等来保证RunLoop不退出
+ 启动RunLoop



本文相关代码github地址 [github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos)

本文参考资料:

[RunLoop官方源码](https://opensource.apple.com/tarballs/CF/)

[iOS底层原理](https://ke.qq.com/course/package/11609)

 	
    