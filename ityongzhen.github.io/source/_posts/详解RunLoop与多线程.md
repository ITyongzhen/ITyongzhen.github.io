---
layout: post
title: 详解RunLoop与多线程
date: 2017-05-15 17:32:24.000000000 +09:00
categories: 
- iOS
---



本文首发于[个人博客](https://ityongzhen.github.io/%E8%AF%A6%E8%A7%A3RunLoop%E4%B8%8E%E5%A4%9A%E7%BA%BF%E7%A8%8B.html)



## 控制线程生命周期（线程保活）

通过上一篇[详解RunLoop之源码分析](https://ityongzhen.github.io/%E8%AF%A6%E8%A7%A3RunLoop%E4%B9%8B%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90.html) 我们知道了runlLoop每条线程都有唯一的一个与之对应的RunLoop对象 并且 RunLoop会在线程结束时销毁
接下来就具体分析RunLoop会在线程结束时销毁这个特点

首先定义定义一个YZThread继承自NSThread 重写它的dealloc方法，为了检测是否销毁
然后，创建该类，并执行方法，如下图所示


~~~~
/** YZThread **/
@interface YZThread : NSThread
@end
@implementation YZThread
- (void)dealloc
{
    NSLog(@"%s", __func__);
}
@end

@interface ViewController ()
@property (strong, nonatomic) YZThread *thread;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
      YZThread *thread = [[YZThread alloc] initWithTarget:self selector:@selector(run) object:nil];
    [thread start];

}
// 这个方法
- (void)run {
     NSLog(@"%s %@", __func__, [NSThread currentThread]);
}

~~~~

运行结果

~~~~
 Interview03-线程保活[7854:134789] -[ViewController run] <YZThread: 0x600000a13b80>{number = 3, name = (null)}
Interview03-线程保活[7854:134789] -[YZThread dealloc]

~~~~
<!--![image.png](https://upload-images.jianshu.io/upload_images/3373351-68ebe0a1db98ffbf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)-->

从结果中，我们发现，当执行完run这个方法之后，该线程就销毁了。

但是在项目中，或者很多第三方中(比如AFN)，可能会遇到这个线程我们经常做事情，如果每次都销毁、创建、销毁、、、那么对性能也是一种损耗.我们就需要我们自己控制线程的销毁和创建。
上面线程之所以立即销毁，根据上篇文章我们知道，

+ 是因为 如果Mode里没有任何Source0/Source1/Timer/Observer，RunLoop会立马退出
那我们给它加上这些东西不就行了么？来试试看吧。

~~~~
/** YZThread **/
@interface YZThread : NSThread
@end
@implementation YZThread
- (void)dealloc
{
    NSLog(@"%s", __func__);
}
@end

@interface ViewController ()
@property (strong, nonatomic) YZThread *thread;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
      YZThread *thread = [[YZThread alloc] initWithTarget:self selector:@selector(run) object:nil];
    [thread start];

}
// 这个方法
- (void)run {
     NSLog(@"%s %@", __func__, [NSThread currentThread]);
    // 往RunLoop里面添加Source\Timer\Observer
    [[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init] forMode:NSDefaultRunLoopMode];
    [[NSRunLoop currentRunLoop] run];
    NSLog(@"%s ----end----", __func__);
}

~~~~

运行结果

~~~~
Interview03-线程保活[7967:136867] -[ViewController run] <YZThread: 0x600003105b00>{number = 3, name = (null)}

~~~~

<!--![image.png](https://upload-images.jianshu.io/upload_images/3373351-fa48f4d18531e917.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)-->
如上所示，添加了port之后，该线程就没有调用dealloc方法了。上一篇介绍了，Source1包含了基于Port的线程间通信，也就是说。添加了port之后，相当于与有了source1 那么线程就不会退出，也就不会调用dealloc方法了。

但是如果我们想在这个线程执行自己的操作，那就需要我们持有这个线程了。如下图所示，增加个属性，并且屏幕点击事件中调用test方法

~~~~

#import "ViewController.h"

/** YZThread **/
@interface YZThread : NSThread
@end
@implementation YZThread
- (void)dealloc
{
    NSLog(@"%s", __func__);
}
@end

@interface ViewController ()
@property (strong, nonatomic) YZThread *thread;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];

    self.thread = [[YZThread alloc] initWithTarget:self selector:@selector(run) object:nil];
    [self.thread start];
}


- (void)run {
    NSLog(@"%s %@", __func__, [NSThread currentThread]);
    // 往RunLoop里面添加Source\Timer\Observer
    [[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init] forMode:NSDefaultRunLoopMode];
    [[NSRunLoop currentRunLoop] run];
    NSLog(@"%s ----end----", __func__);
}

// 这个方法的目的：线程保活
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    NSLog(@"点击屏幕");
    [self performSelector:@selector(test) onThread:self.thread withObject:nil waitUntilDone:NO];
}
~~~~

运行结果：

~~~~
Interview03-线程保活[8035:138457] -[ViewController run] <YZThread: 0x60000232d4c0>{number = 3, name = (null)}
 Interview03-线程保活[8035:138399] 点击屏幕
Interview03-线程保活[8035:138457] -[ViewController test] <YZThread: 0x60000232d4c0>{number = 3, name = (null)}
~~~~
<!--![image.png](https://upload-images.jianshu.io/upload_images/3373351-28907f837de1264c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)-->

果然是线程不会销毁，而且 由 number = 3 可知，确实是在我们自己创建的线程中执行的操作

接下来我们再看一个问题，首先在控制器中写上dealloc方法，

~~~~
- (void)dealloc
{
    NSLog(@"%s", __func__);
    
}

~~~~
然后给个导航控制器跳转到当前控制器，之后返回的时候发现dealloc并没有执行
这是因为代码中用的

~~~~
 self.thread = [[YZThread alloc] initWithTarget:self selector:@selector(run) object:nil];
~~~~
这个方法，会导致线程持有当前控制器，导致循环引用，如果解除循环引用，需要把这个initWithTarget 换成initWithBlock

我们将之前的

~~~~
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.thread = [[YZThread alloc] initWithTarget:self selector:@selector(run) object:nil];
    [self.thread start];
}
- (void)run {
    NSLog(@"%s %@", __func__, [NSThread currentThread]);
    // 往RunLoop里面添加Source\Timer\Observer
    [[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init] forMode:NSDefaultRunLoopMode];
    [[NSRunLoop currentRunLoop] run];
    NSLog(@"%s ----end----", __func__);
}
~~~~
改成

~~~~

- (void)viewDidLoad {
    [super viewDidLoad];
    
   self.thread = [[MJThread alloc] initWithBlock:^{
        NSLog(@"%@----begin----", [NSThread currentThread]);
        
        // 往RunLoop里面添加Source\Timer\Observer
        [[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init] forMode:NSDefaultRunLoopMode];
        // 这个方法会不能退出循环
        [[NSRunLoop currentRunLoop] run];
               
    }];
    [self.thread start];
}

~~~~

可以发现

~~~~
Interview03-线程保活[8035:138457] -[ViewController dealloc]
~~~~
<!--![image.png](https://upload-images.jianshu.io/upload_images/3373351-7a5b894d4017485f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)-->

这次控制器可以销毁了。然后，你可能已经发现了，虽然控制器可以销毁，但是线程并没有销毁。换句话说，虽然没有循环引用了。而且控制器销毁了，然后线程依然没有销毁，这是为什么呢。
### 注意[[NSRunLoop currentRunLoop] run]
其实关键的一句代码是
~~~~
[[NSRunLoop currentRunLoop] run];
~~~~

看一下官方对这个方法的解释

~~~~
Discussion

If no input sources or timers are attached to the run loop, this method exits immediately;
 otherwise, it runs the receiver in the NSDefaultRunLoopMode by repeatedly invoking runMode:beforeDate:. 
 In other words, this method effectively begins an infinite loop that processes data from the run loop’s input sources and timers.
~~~~
也就是说，NSRunLoop的run方法是无法停止的，它专门用于开启一个永不销毁的线程（NSRunLoop）
就相当于

~~~~
while (1) {
            [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        }
        
~~~~
就算我们手动调用停止RunLoop的方法，

~~~~
// 停止RunLoop
CFRunLoopStop(CFRunLoopGetCurrent());
    
~~~~
也无法停止这个线程，因为这个只能停止这一次的RunLoop,下次循环，依然可以继续进行下去

我们把代码修改为如下的写法

~~~~
- (void)viewDidLoad {
    [super viewDidLoad];
    
    __weak typeof(self) weakSelf = self;
      
    self.stopped = NO;
    self.thread = [[MJThread alloc] initWithBlock:^{
        NSLog(@"%@----begin----", [NSThread currentThread]);
        
        // 往RunLoop里面添加Source\Timer\Observer
        [[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init] forMode:NSDefaultRunLoopMode];
    
        while (weakSelf && !weakSelf.isStoped) {
            [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        }
        
        NSLog(@"%@----end----", [NSThread currentThread]);
    }];
    [self.thread start];
}

// 用于停止子线程的RunLoop
- (void)stopThread
{
    // 设置标记为NO
    self.stopped = YES;
    
    // 停止RunLoop
    CFRunLoopStop(CFRunLoopGetCurrent());
    NSLog(@"%s %@", __func__, [NSThread currentThread]);
}

~~~~

我们用一个bool值做未标记,当下次进来的时候。判断已经停止了。那就不进入下次RunLoop了，就可以退出循环

**注意**

~~~~
// 在子线程调用stop（waitUntilDone设置为YES，代表子线程的代码执行完毕后，这个方法才会往下走）
    [self performSelector:@selector(stopThread) onThread:self.thread withObject:nil waitUntilDone:YES];
~~~~

至此，我们代码如下所示

~~~~
#import "ViewController.h"
#import "MJThread.h"

@interface ViewController ()
@property (strong, nonatomic) MJThread *thread;
@property (assign, nonatomic, getter=isStoped) BOOL stopped;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    __weak typeof(self) weakSelf = self;
    
    self.stopped = NO;
    
    self.thread = [[MJThread alloc] initWithBlock:^{
        NSLog(@"%@----begin----", [NSThread currentThread]);
        // 往RunLoop里面添加Source\Timer\Observer
        [[NSRunLoop currentRunLoop] addPort:[[NSPort alloc] init] forMode:NSDefaultRunLoopMode];
        // 这个方法会不能退出循环
//        [[NSRunLoop currentRunLoop] run];
        while (!weakSelf.isStoped) {
            [[NSRunLoop currentRunLoop] runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        }
        NSLog(@"%@----end----", [NSThread currentThread]);

        
    }];
    [self.thread start];
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    [self performSelector:@selector(test) onThread:self.thread withObject:nil waitUntilDone:NO];
}

// 子线程需要执行的任务
- (void)test
{
    NSLog(@"%s %@", __func__, [NSThread currentThread]);
}

- (IBAction)stop {
    // 在子线程调用stop
    [self performSelector:@selector(stopThread) onThread:self.thread withObject:nil waitUntilDone:NO];
}

// 用于停止子线程的RunLoop
- (void)stopThread
{
    // 设置标记为NO
    self.stopped = YES;
    
    // 停止RunLoop
    CFRunLoopStop(CFRunLoopGetCurrent());
    NSLog(@"%s %@", __func__, [NSThread currentThread]);
}

- (void)dealloc
{
    NSLog(@"%s", __func__);
    
//    [self stop];
}

~~~~

<!--![image.png](https://upload-images.jianshu.io/upload_images/3373351-96f84c22b8766692.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)-->

### 封装线程
到这里，我们已经可以控制线程的创建，执行，销毁了。
但是，这也太复杂了吧。每次我们都需要写这么多代码，如果我们每次，都这么干的话，岂不是疯掉了。所以我们要抽取出来，这样，以后用的时候，直接拿来就可以了。

首先新建一个继承自 NSObject的类MJPermenantThread 然后持有一个NSThread的属性，注意这里不是直接继承NSThread 是因为，如果继承了NSThread 那么外界调用的时候，可以直接使用NSThread的方法，会影响线程的使用。所以，本着简洁，可控的原则，定义的类继承自NSThread 然后拥有属性NSThread，这样，外界只能使用我们暴露出去接口。

.h中代码如下

~~~~
typedef void (^MJPermenantThreadTask)(void);

@interface MJPermenantThread : NSObject

/**
 开启线程
 */
//- (void)run;

/**
 在当前子线程执行一个任务
 */
- (void)executeTask:(MJPermenantThreadTask)task;

/**
 结束线程
 */
- (void)stop;



~~~~
.m中代码如下

~~~~
/** MJPermenantThread **/
@interface MJPermenantThread()
@property (strong, nonatomic) NSThread *innerThread;
@end

@implementation MJPermenantThread
#pragma mark - public methods
- (instancetype)init
{
    if (self = [super init]) {
        self.innerThread = [[NSThread alloc] initWithBlock:^{
            NSLog(@"begin----");
            
            // 创建上下文（要初始化一下结构体）
            CFRunLoopSourceContext context = {0};
            
            // 创建source
            CFRunLoopSourceRef source = CFRunLoopSourceCreate(kCFAllocatorDefault, 0, &context);
            
            // 往Runloop中添加source
            CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopDefaultMode);
            
            // 销毁source
            CFRelease(source);
            
            // 启动
            CFRunLoopRunInMode(kCFRunLoopDefaultMode, 1.0e10, false);
            
            //            while (weakSelf && !weakSelf.isStopped) {
            //                // 第3个参数：returnAfterSourceHandled，设置为true，代表执行完source后就会退出当前loop
            //                CFRunLoopRunInMode(kCFRunLoopDefaultMode, 1.0e10, true);
            //            }
            
            NSLog(@"end----");
        }];
        
        [self.innerThread start];
    }
    return self;
}

//- (void)run
//{
//    if (!self.innerThread) return;
//
//    [self.innerThread start];
//}

- (void)executeTask:(MJPermenantThreadTask)task
{
    if (!self.innerThread || !task) return;
    
    [self performSelector:@selector(__executeTask:) onThread:self.innerThread withObject:task waitUntilDone:NO];
}

- (void)stop
{
    if (!self.innerThread) return;
    
    [self performSelector:@selector(__stop) onThread:self.innerThread withObject:nil waitUntilDone:YES];
}

- (void)dealloc
{
    NSLog(@"%s", __func__);
    
    [self stop];
}

#pragma mark - private methods
- (void)__stop
{
    CFRunLoopStop(CFRunLoopGetCurrent());
    self.innerThread = nil;
}

- (void)__executeTask:(MJPermenantThreadTask)task
{
    task();
}

@end


~~~~

MJPermenantThread.h 就是这样的


~~~~
#import <Foundation/Foundation.h>

typedef void (^MJPermenantThreadTask)(void);

@interface MJPermenantThread : NSObject

/**
 开启线程
 */
//- (void)run;

/**
 在当前子线程执行一个任务
 */
- (void)executeTask:(MJPermenantThreadTask)task;

/**
 结束线程
 */
- (void)stop;

@end

~~~~

<!--![image.png](https://upload-images.jianshu.io/upload_images/3373351-92bf3f5f5c91efcd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)-->
MJPermenantThread.m 就是这样的

~~~~
#import "MJPermenantThread.h"

/** MJThread **/
@interface MJThread : NSThread
@end
@implementation MJThread
- (void)dealloc
{
    NSLog(@"%s", __func__);
}
@end

/** MJPermenantThread **/
@interface MJPermenantThread()
@property (strong, nonatomic) NSThread *innerThread;
@end

@implementation MJPermenantThread
#pragma mark - public methods
- (instancetype)init
{
    if (self = [super init]) {
        self.innerThread = [[NSThread alloc] initWithBlock:^{
            NSLog(@"begin----");
            
            // 创建上下文（要初始化一下结构体）
            CFRunLoopSourceContext context = {0};
            
            // 创建source
            CFRunLoopSourceRef source = CFRunLoopSourceCreate(kCFAllocatorDefault, 0, &context);
            
            // 往Runloop中添加source
            CFRunLoopAddSource(CFRunLoopGetCurrent(), source, kCFRunLoopDefaultMode);
            
            // 销毁source
            CFRelease(source);
            
            // 启动
            CFRunLoopRunInMode(kCFRunLoopDefaultMode, 1.0e10, false);
            
            //            while (weakSelf && !weakSelf.isStopped) {
            //                // 第3个参数：returnAfterSourceHandled，设置为true，代表执行完source后就会退出当前loop
            //                CFRunLoopRunInMode(kCFRunLoopDefaultMode, 1.0e10, true);
            //            }
            
            NSLog(@"end----");
        }];
        
        [self.innerThread start];
    }
    return self;
}

//- (void)run
//{
//    if (!self.innerThread) return;
//
//    [self.innerThread start];
//}

- (void)executeTask:(MJPermenantThreadTask)task
{
    if (!self.innerThread || !task) return;
    
    [self performSelector:@selector(__executeTask:) onThread:self.innerThread withObject:task waitUntilDone:NO];
}

- (void)stop
{
    if (!self.innerThread) return;
    
    [self performSelector:@selector(__stop) onThread:self.innerThread withObject:nil waitUntilDone:YES];
}

- (void)dealloc
{
    NSLog(@"%s", __func__);
    
    [self stop];
}

#pragma mark - private methods
- (void)__stop
{
    CFRunLoopStop(CFRunLoopGetCurrent());
    self.innerThread = nil;
}

- (void)__executeTask:(MJPermenantThreadTask)task
{
    task();
}

@end

~~~~

<!--![image.png](https://upload-images.jianshu.io/upload_images/3373351-21d4500bcdd19d2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![image.png](https://upload-images.jianshu.io/upload_images/3373351-d7651f5badd92a1f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)-->

这样就简单封装了线程，而且因为是用C语言创建的，能更灵活的控制 看官方对CFRunLoopAddSource的定义，第三个参数，如果是false,那么这个线程，执行完当前这次RunLoop，不会退出。这样写的话，就可以保证线程一直存在。

~~~~

 //第三个参数 ：A flag indicating whether the run loop should exit after processing one source. 
 //If false, the run loop continues processing events until seconds has passed.
 CFRunLoopAddSource(CFRunLoopRef rl, CFRunLoopSourceRef source, CFRunLoopMode mode);
~~~~

### 封装之后的使用
使用的时候，我们可以主动调用stop 来销毁掉线程，也可以，不做任何处理，当控制器销毁时候，MJPermenantThread对象就会销毁，根据

~~~~
- (void)dealloc
{
    NSLog(@"%s", __func__);
    
    [self stop];
}
- (void)stop
{
    if (!self.innerThread) return;
    
    [self performSelector:@selector(__stop) onThread:self.innerThread withObject:nil waitUntilDone:YES];
}

- (void)__stop
{
    CFRunLoopStop(CFRunLoopGetCurrent());
    self.innerThread = nil;
}
~~~~
可知，RunLoop也会停止，线程也就退出了。所以使用的时候，如下所示

~~~~
#import "ViewController.h"
#import "MJPermenantThread.h"

@interface ViewController ()
@property (strong, nonatomic) MJPermenantThread *thread;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
     self.thread = [[MJPermenantThread alloc] init];
       
}

- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event
{
    [self.thread executeTask:^{
        NSLog(@"执行任务 - %@", [NSThread currentThread]);
    }];
    
}

- (IBAction)stop {
    [self.thread stop];
}

- (void)dealloc
{
    NSLog(@"%s", __func__);
}
@end
~~~~
<!--![image.png](https://upload-images.jianshu.io/upload_images/3373351-638ce4f6cd8239dc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)-->

至此。就说完了RunLoop和线程直接关系，以及线程的保活，以及自定义线程。这样使用起来更加得心用手。


本文相关代码github地址 [github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos)

本文参考资料:

[RunLoop官方源码](https://opensource.apple.com/tarballs/CF/)

[iOS底层原理](https://ke.qq.com/course/package/11609)

