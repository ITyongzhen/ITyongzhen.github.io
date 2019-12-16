---
layout: post
title: 详解RunLoop之源码分析
date: 2017-04-08 17:32:24.000000000 +09:00
categories: 
- iOS
---

 首发于 [个人博客](https://ityongzhen.github.io/%E8%AF%A6%E8%A7%A3RunLoop%E4%B9%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.html)
## RunLoop是什么
runloop 是什么？Runloop 还是比较顾名思义的一个东西，说白了就是一种循环，只不过它这种循环比较高级。一般的 while 循环会导致 CPU 进入忙等待状态，而 Runloop 则是一种“闲”等待，这部分可以类比 Linux 下的 epoll。当没有事件时，Runloop 会进入休眠状态，有事件发生时， Runloop 会去找对应的 Handler 处理事件。Runloop 可以让线程在需要做事的时候忙起来，不需要的话就让线程休眠

### 开始之前，先想想这几道面试题
+ runloop和线程的关系？
+ timer 与 runloop 的关系？
+ 程序中添加每3秒响应一次的NSTimer，当拖动tableview时timer可能无法响应要怎么解决？
+ runloop内部实现逻辑？
+ runloop 是怎么响应用户操作的， 具体流程是什么样的？
+ 说说runLoop的几种状态
+ runloop的mode作用是什么？

## 源码分析
回答问题之前，我们先看源码

RunLoop 源码  https://opensource.apple.com/tarballs/CF/ 里面数字最大的是最 新的，下载最新的 	CF-1153.18.tar.gz(写本文时候的最新版本)
 
查看源码 中的

 ````
 CFRunLoopRef CFRunLoopGetCurrent(void) {
       CHECK_FOR_FORK();
       CFRunLoopRef rl = (CFRunLoopRef)_CFGetTSD(__CFTSDKeyRunLoop);
       if (rl) return rl;
       return _CFRunLoopGet0(pthread_self());
}
````
通过 
````
	_CFRunLoopGet0 获取的
````


进去查看做了什么

````
loop = (CFRunLoopRef)CFDictionaryGetValue(__CFRunLoops, pthreadPointer(t));
	if (!loop) {
	    CFDictionarySetValue(__CFRunLoops, pthreadPointer(t), newLoop);
	    loop = newLoop;
	}
````

发现有这么一个获取线程的方法，也就是传入一个线程作为key，获取一个loop,如果loop为空，就以这个线程为key创建runloop

小结:

+ 每条线程都有唯一的一个与之对应的RunLoop对象
+ RunLoop保存在一个全局的Dictionary里，线程作为key，RunLoop作为value
+ 线程刚创建时并没有RunLoop对象，RunLoop会在第一次获取它时创建
+ RunLoop会在线程结束时销毁(下面分析这一条)

### runloop的mode
接下来认识一下runloop的主要类
Core Foundation中关于RunLoop的5个类

````
CFRunLoopRef

CFRunLoopModeRef

CFRunLoopSourceRef

CFRunLoopTimerRef

CFRunLoopObserverRef
````
看一下runloop结构体

````
struct __CFRunLoop {
    CFRuntimeBase _base;
    pthread_mutex_t _lock;			/* locked for accessing mode list */
    __CFPort _wakeUpPort;			// used for CFRunLoopWakeUp 
    Boolean _unused;
    volatile _per_run_data *_perRunData;              // reset for runs of the run loop
    pthread_t _pthread;
    uint32_t _winthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode;
    CFMutableSetRef _modes;
    struct _block_item *_blocks_head;
    struct _block_item *_blocks_tail;
    CFAbsoluteTime _runTime;
    CFAbsoluteTime _sleepTime;
    CFTypeRef _counterpart;
};
````
只保留主要的就剩下了

````
typedef struct __CFRunLoop * CFRunLoopRef;
 struct __CFRunLoop {
	 pthread_t _pthread;
    CFMutableSetRef _commonModes;
    CFMutableSetRef _commonModeItems;
    CFRunLoopModeRef _currentMode; //当前模式
    CFMutableSetRef _modes; //所有的模式
  	CFMutableSetRef _modes;
};
````
理解为CFRunLoopRef中包含有_modes，modes是由 CFRunLoopModeRef组成的集合
这些modes中，只有一种是当前模式，称为 _currentMode

接下来我们看看runloopmode中究竟有什么，同样，只保留主要的，关键就是下面4个

总结起来就是

+ CFRunLoopModeRef代表RunLoop的运行模式
+ 一个RunLoop包含若干个Mode，每个Mode又包含若干个Source0/Source1/Timer/Observer
+ RunLoop启动时只能选择其中一个Mode，作为currentMode
+ 如果需要切换Mode，只能退出当前Loop，再重新选择一个Mode进入
+ 不同组的Source0/Source1/Timer/Observer能分隔开来，互不影响
+ 如果Mode里没有任何Source0/Source1/Timer/Observer，RunLoop会立马退出


````
typedef struct __CFRunLoopMode *CFRunLoopModeRef;
struct __CFRunLoopMode {

    CFMutableSetRef _sources0;
    CFMutableSetRef _sources1;
    CFMutableArrayRef _observers;
    CFMutableArrayRef _timers;
};
````
我们可以理解为，RunLoop中有许多模式，但当前运行的只有一种，一个图来表示，就是
![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf475b92b1a?w=374&h=241&f=png&s=18430)

具体在某一种runloop中的运行逻辑，官方给出下图
![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf4758fcfda?w=758&h=393&f=png&s=115227)

那么，前面说的，_sources0、_sources1、_observers、_timers J究竟包含了什么呢？
先用一张图来总结一下，然后再详细介绍
![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf4914f7d6b?w=1008&h=484&f=png&s=143230)

如上图所示，_sources0 包含触摸事件，和 performSelector:onThread:
跑一下代码证明一下，
首先创建一个新项目，实现点击事件，NSLog只是为了打断点

````
#import "ViewController.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
}

-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    NSLog(@"这个打印只是为了打断点");
}

@end

断点暂停之后，输入lldb指令 bt 之后如图所示


````
![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf4916b9b70?w=1202&h=674&f=png&s=373644)

从打印日志来看 调用了__CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ 这个SOURCE0方法
那么接下来验证一下performSelector
![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf496a29275?w=1240&h=637&f=png&s=421741)
由上图可知，performSelector 也是执行了source0

那我们再看一下Timer
![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf54db87be7?w=1240&h=645&f=png&s=473496)
如上图所示，这次是__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__

对于其他几种情况，读者课自行验证。


用一幅图来总结RunLoop的运行逻辑
![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf4b87b6e40?w=962&h=489&f=png&s=130629)

## 源码内部细节分析
要想分析源码首先要知道入口在哪里，由前面的断点可知
![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf4cf942711?w=838&h=288&f=png&s=144993)
入口为 CFRunLoopRunSpecific
去源码中找到之后发现有很多。只保留关键信息

````

SInt32 CFRunLoopRunSpecific(CFRunLoopRef rl, CFStringRef modeName, CFTimeInterval seconds, Boolean returnAfterSourceHandled) {     /* DOES CALLOUT */
    CHECK_FOR_FORK();
    // 通知Observers: 进入Loop
  __CFRunLoopDoObservers(rl, currentMode, kCFRunLoopEntry);
    // 具体要做的事情
	result = __CFRunLoopRun(rl, currentMode, seconds, returnAfterSourceHandled, previousMode);
    
     // 通知Observers: 退出Loop
	__CFRunLoopDoObservers(rl, currentMode, kCFRunLoopExit);
    
    return result;
}
````
那我们继续跟__CFRunLoopRun 看看做了什么，发现里面很长的东西，整理了一下，只保留关键代码，如下

````
/* rl, rlm are locked on entrance and exit */
static int32_t __CFRunLoopRun(CFRunLoopRef rl, CFRunLoopModeRef rlm, CFTimeInterval seconds, Boolean stopAfterHandle, CFRunLoopModeRef previousMode) {
    int32_t retVal = 0;
    do {
        // 通知Observers: 即将处理Timers
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeTimers);
        // 通知Observers: 即将处理Sources
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeSources);
        // 处理Blocks
        __CFRunLoopDoBlocks(rl, rlm);
        // 处理Sources0
        if (__CFRunLoopDoSources0(rl, rlm, stopAfterHandle)) {
            // 处理Blocks
            __CFRunLoopDoBlocks(rl, rlm);
        }
        Boolean poll = sourceHandledThisLoop || (0ULL == timeout_context->termTSR);
        //判断有无source1
        if (__CFRunLoopServiceMachPort(dispatchPort, &msg, sizeof(msg_buffer), &livePort, 0, &voucherState, NULL)) { // 如果有source1 就跳转到 handle_msg
            goto handle_msg;
        }
        // 通知Observers: 即将休眠
        __CFRunLoopDoObservers(rl, rlm, kCFRunLoopBeforeWaiting);
        __CFRunLoopSetSleeping(rl);
        // 通知Observers: 结束休眠
        __CFRunLoopServiceMachPort(waitSet, &msg, sizeof(msg_buffer), &livePort, poll ? 0 : TIMEOUT_INFINITY, &voucherState, &voucherCopy);
    handle_msg:;
        
        if (被Timer唤醒) {
            // 处理Timers
            __CFRunLoopDoTimers(rl, rlm, mach_absolute_time())
        } else if (被gcd唤醒) {
            //处理GCD
            __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__(msg);
        } else { //能来到这里，就说明被Source1唤醒
            
            __CFRunLoopDoSource1(rl, rlm, rls, msg, msg->msgh_size, &reply) || sourceHandledThisLoop;
            // 处理Blocks
            __CFRunLoopDoBlocks(rl, rlm);
            
            // 设置返回值，决定是否继续循环
            if (sourceHandledThisLoop && stopAfterHandle) {
                retVal = kCFRunLoopRunHandledSource;
            } else if (timeout_context->termTSR < mach_absolute_time()) {
                retVal = kCFRunLoopRunTimedOut;
            } else if (__CFRunLoopIsStopped(rl)) {
                __CFRunLoopUnsetStopped(rl);
                retVal = kCFRunLoopRunStopped;
            } else if (rlm->_stopped) {
                rlm->_stopped = false;
                retVal = kCFRunLoopRunStopped;
            } else if (__CFRunLoopModeIsEmpty(rl, rlm, previousMode)) {
                retVal = kCFRunLoopRunFinished;
            }
            voucher_mach_msg_revert(voucherState);
            os_release(voucherCopy);
        } while (0 == retVal);
        
        return retVal;
}


````

截图下来的话，就是这样的
![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf4b9d8cd53?w=1240&h=803&f=png&s=361091)

###调用细节
前面说了大概的流程，那么，具体怎么调用的呢，lldb调试堆栈的时候，那些方法怎么调用的呢？这里以 __CFRunLoopDoTimers 为例，看下源码怎么调用的

![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf4bcf9ab23?w=1240&h=478&f=png&s=230580)
上图可知，关键代码是 CFRunLoopTimerRef
继续查看 CFRunLoopTimerRef

![image.png](https://user-gold-cdn.xitu.io/2019/6/15/16b5aaf4cc507cd3?w=1240&h=835&f=png&s=379870)
关键代码是__CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__
看到这里是不是对前面截图中的调用堆栈更清晰了呢。
其他几种也都是类似的逻辑，就不赘述了。


## 目前已知的Mode有五种
````
目前已知的Mode有5种
kCFRunLoopDefaultMode：App的默认Mode，通常主线程是在这个Mode下运行

UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响

UIInitializationRunLoopMode：在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用

GSEventReceiveRunLoopMode：接受系统事件的内部 Mode，通常用不到

kCFRunLoopCommonModes：这是一个占位用的Mode，不是一种真正的Mode
````

## RunLoop的状态

````
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),           // 即将进入Loop
    kCFRunLoopBeforeTimers = (1UL << 1),    //即将处理Timer
    kCFRunLoopBeforeSources = (1UL << 2),   //即将处理Source
    kCFRunLoopBeforeWaiting = (1UL << 5),   //即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),    //刚从休眠中唤醒
    kCFRunLoopExit = (1UL << 7),            //即将退出Loop
    kCFRunLoopAllActivities = 0x0FFFFFFFU
};
````


常见的2种Mode

+ kCFRunLoopDefaultMode（NSDefaultRunLoopMode）：App的默认Mode，通常主线程是在这个Mode下运行

+ UITrackingRunLoopMode：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响


## RunLoop 和 NSTimer
我们知道，默认情况下，NSTimer计时器，会被UIScrollView 打断，会影响计时器的使用。原因就是滚动时候，RunLoop切换到了UITrackingRunLoopMode模式下，但计时器在NSDefaultRunLoopMode下，所以就停止了。解决办法就是设置NSRunLoopCommonModes。特别注意的是：

**NSRunLoopCommonModes并不是一个真的模式，它只是一个标记**


本文参考资料:

[RunLoop官方源码](https://opensource.apple.com/tarballs/CF/)

[iOS底层原理](https://ke.qq.com/course/package/11609)




