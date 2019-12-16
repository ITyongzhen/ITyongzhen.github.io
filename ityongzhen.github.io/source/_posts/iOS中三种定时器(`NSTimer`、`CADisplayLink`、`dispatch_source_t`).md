---
layout: post
title: iOS中三种定时器(`NSTimer`、`CADisplayLink`、`dispatch_source_t`)
date: 2017-12-15 15:32:24.000000000 +09:00
categories: 
- iOS
---

## 前言
iOS中定时器的使用还是很常见的。那么iOS中有几种定时器，平时又怎么使用呢？

### NSTimer
在详解RunLoop之源码分析一文中，简单描述了NStimer和RunLoop的关系

> 默认情况下，NSTimer计时器，会被UIScrollView 打断，会影响计时器的使用。原因就是滚动时候，RunLoop切换到了UITrackingRunLoopMode模式下，但计时器在NSDefaultRunLoopMode下，所以就停止了。解决办法就是设置NSRunLoopCommonModes。特别注意的是：NSRunLoopCommonModes并不是一个真的模式，它只是一个标记.如果设置了NSRunLoopCommonModes timer能在_commonModes数组中存放的模式下工作。

## 使用
下面两个定时器的使用是等价的

~~~~
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(test) userInfo:nil repeats:YES];
    
self.timer = [NSTimer timerWithTimeInterval:1.0 target:self selector:@selector(test) userInfo:nil repeats:YES];
[[NSRunLoop currentRunLoop] addTimer:self.timer forMode:NSRunLoopCommonModes];
~~~~

上面两个方法的等价的，区别是第二张，需要自己手动加到RunLoop中，否则不生效。苹果中关于NSTimer的源码是不开源的，但是我们可以参考[GNUstep源码地址](http://www.gnustep.org/resources/downloads.php)中的源码

### 	scheduledTimerWithTimeInterval

~~~~
	+ (NSTimer*) scheduledTimerWithTimeInterval: (NSTimeInterval)ti
					     target: (id)object
					   selector: (SEL)selector
					   userInfo: (id)info
					    repeats: (BOOL)f
	{
	  id t = [[self alloc] initWithFireDate: nil
				       interval: ti
					 target: object
				       selector: selector
				       userInfo: info
					repeats: f];
	  [[NSRunLoop currentRunLoop] addTimer: t forMode: NSDefaultRunLoopMode];
	  RELEASE(t);
	  return t;
	}
~~~~

### 和 timerWithTimeInterval

~~~~
	+ (NSTimer*) timerWithTimeInterval: (NSTimeInterval)ti
				    target: (id)object
				  selector: (SEL)selector
				  userInfo: (id)info
				   repeats: (BOOL)f
	{
	  return AUTORELEASE([[self alloc] initWithFireDate: nil
						   interval: ti
						     target: object
						   selector: selector
						   userInfo: info
						    repeats: f]);
	}
~~~~

从上面的源码可知，这两种方式，调用的定时器是一样的，但是第一种会自动添加到RunLoop中，不需要我们来处理了。

但是上面两种都会导致循环引用。原因也很好理解，控制器持有定时器，定时器的target指向当前控制器，所以就循环引用了。

## 解决循环引用
**__weak 不能解除循环引用**

~~~~
	 __weak typeof(self) weakSelf = self;
	    
	self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target: weakSelf selector:@selector(test) userInfo:nil repeats:YES];
~~~~	
	
__weak 和block 能解除循环引用

~~~~
__weak typeof(self) weakSelf = self;
 
self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 repeats:YES block:^(NSTimer * _Nonnull timer) {
        [weakSelf test];
}];
~~~~

上面的代码中是可以解除循环引用的，然后真正起作用的，是block。和timer并没有加什么关系，详细可以看[深入理解iOS的block](https://ityongzhen.github.io/%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3iOS%E7%9A%84block.html)一文，有详细说明。

那么问题来了，要怎么解除循环引用呢？

**invalidate解除循环引用**

~~~~
	- (void)viewDidLoad {
	    [super viewDidLoad];
		self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(test) userInfo:nil repeats:YES];
	}
	
	
	- (void)viewWillDisappear:(BOOL)animated{
	    [super viewWillDisappear:animated];
	    [self.timer invalidate];
	}
~~~~
	
	
上面代码中，在控制器即将消失的时候，调用`[self.timer invalidate]`;能解除循环引用。但是，在开发中一般不这样用。因为，页面跳转了就会调用`viewWillDisappear`,然后有时候业务逻辑很复杂，此时并不想取消定时器。更多的时候想让定时器和控制器的生命绑定在一起，那我们可否这么写呢

~~~~
- (void)viewDidLoad {
    [super viewDidLoad];
	self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:self selector:@selector(test) userInfo:nil repeats:YES];
}

-(void)dealloc{
     NSLog(@"%s",__func__);
    // 写在这里没用
//   [self.timer invalidate];
}
~~~~

答案是不行的，因为已经循环引用了，在`dealloc`里面调用`[self.timer invalidate]`，那这代码永远不会执行。

- `NSProxy`解除循环引用

`NSProxy`是不继承自`NSObject`的。专门用来做这个事的。

`NSProxy`的效率很高，因为不经过`Runtime`的，消息发送，消息动态解析，去缓存中查找等流程，直接通过消息转发。关于`Runtime`的详细分析，可以参考详解iOS中的`Runtime`

API如下

~~~~
@interface NSProxy <NSObject> {
    Class	isa;
}

+ (id)alloc;
+ (id)allocWithZone:(nullable NSZone *)zone NS_AUTOMATED_REFCOUNT_UNAVAILABLE;
+ (Class)class;

- (void)forwardInvocation:(NSInvocation *)invocation;
- (nullable NSMethodSignature *)methodSignatureForSelector:(SEL)sel NS_SWIFT_UNAVAILABLE("NSInvocation and related APIs not available");
- (void)dealloc;
- (void)finalize;
@property (readonly, copy) NSString *description;
@property (readonly, copy) NSString *debugDescription;
+ (BOOL)respondsToSelector:(SEL)aSelector;

- (BOOL)allowsWeakReference NS_UNAVAILABLE;
- (BOOL)retainWeakReference NS_UNAVAILABLE;

// - (id)forwardingTargetForSelector:(SEL)aSelector;

@end
~~~~

例如isKindOfClass和isMemberOfClass等等，都是直接走消息转发

~~~~
- (BOOL) isKindOfClass: (Class)aClass
{
  NSMethodSignature	*sig;
  NSInvocation		*inv;
  BOOL			ret;

  sig = [self methodSignatureForSelector: _cmd];
  inv = [NSInvocation invocationWithMethodSignature: sig];
  [inv setSelector: _cmd];
  [inv setArgument: &aClass atIndex: 2];
  [self forwardInvocation: inv];
  [inv getReturnValue: &ret];
  return ret;
}


- (BOOL) isMemberOfClass: (Class)aClass
{
  NSMethodSignature	*sig;
  NSInvocation		*inv;
  BOOL			ret;

  sig = [self methodSignatureForSelector: _cmd];
  inv = [NSInvocation invocationWithMethodSignature: sig];
  [inv setSelector: _cmd];
  [inv setArgument: &aClass atIndex: 2];
  [self forwardInvocation: inv];
  [inv getReturnValue: &ret];
  return ret;
}

~~~~


新建类`YZProxy`继承自`NSProxy`,具体代码如下

~~~~
#import <Foundation/Foundation.h>

@interface YZProxy : NSProxy
+ (instancetype)proxyWithTarget:(id)target;
@property (weak, nonatomic) id target;
@end
#import "YZProxy.h"

@implementation YZProxy

+ (instancetype)proxyWithTarget:(id)target
{
    // NSProxy对象不需要调用init，因为它本来就没有init方法
    YZProxy *proxy = [YZProxy alloc];
    proxy.target = target;
    return proxy;
}

- (NSMethodSignature *)methodSignatureForSelector:(SEL)sel
{
    return [self.target methodSignatureForSelector:sel];
}

- (void)forwardInvocation:(NSInvocation *)invocation
{
    [invocation invokeWithTarget:self.target];
}
@end
使用的时候，如下就可以了。

- (void)viewDidLoad {
    [super viewDidLoad];
	self.timer = [NSTimer scheduledTimerWithTimeInterval:1.0 target:[YZProxy proxyWithTarget:self] selector:@selector(test) userInfo:nil repeats:YES];
}


- (void)dealloc{
    [self.timer invalidate];
}
~~~~

## CADisplayLink
除了NSTimer之外，还可以使用CADisplayLink定时器

~~~~
- (void)viewDidLoad {
    [super viewDidLoad];
	 // 调用频率和屏幕的刷新帧率一致，60FPS
    self.link = [CADisplayLink displayLinkWithTarget:self selector:@selector(test)];
    [self.link addToRunLoop:[NSRunLoop currentRunLoop]  forMode:NSDefaultRunLoopMode];
}


- (void)dealloc{
    [self.timer invalidate];
}
~~~~

### 注意点
`CADisplayLink`和`NSTimer`一样也会导致循环引用，解决办法和前面的`NSTimer`一样。区别就是`CADisplayLink`并没有类似`NSTimer`中的`block `方法` ：+ (NSTimer *)scheduledTimerWithTimeInterval:(NSTimeInterval)interval repeats:(BOOL)repeats block:(void (^)(NSTimer *timer))block`

### dispatch_source_t

前面说了定时器`NSTimer`和`CADisplayLink`,但是，他们都是和`RunLoop`相关的，所以，从详解RunLoop之源码分析中我们知道，我们的计时器设置是每1秒执行一次，假设`RunLoop`执行完一圈耗时0.3秒，当执行0.8秒的时候，开始下一圈的`RunLoop`，当执行完之后，已经是1.1秒了。所以，这两种计时器不够精确。当然了，实际上RunLoop每一圈的耗时远远小于0.3，这里只是为了方便说明问题而举例。

~~~~
@interface ViewController ()
@property (strong, nonatomic) dispatch_source_t GCDtimer;
@end

@implementation ViewController


- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    self.view.backgroundColor = [UIColor yellowColor];
    [self GCDTest];
    
}


- (void)GCDTest
{
    
    // 主队列
    //    dispatch_queue_t queue = dispatch_get_main_queue();
    
    // 创建一个队列
    dispatch_queue_t queue = dispatch_queue_create("timer", DISPATCH_QUEUE_SERIAL);
    
    // 创建定时器
    dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
    
    // 设置时间
    uint64_t start = 2.0; // 2秒后开始执行
    uint64_t interval = 1.0; // 每隔1秒执行
    dispatch_source_set_timer(timer,
                              dispatch_time(DISPATCH_TIME_NOW, start * NSEC_PER_SEC),
                              interval * NSEC_PER_SEC, 0);
    
    // 设置回调
    //    dispatch_source_set_event_handler(timer, ^{
    //        NSLog(@"1111");
    //    });
    dispatch_source_set_event_handler_f(timer, timerFire);
    
    // 启动定时器
    dispatch_resume(timer);
    
    self.GCDtimer = timer;
}

void timerFire(void *param)
{
    NSLog(@"2222 - %@", [NSThread currentThread]);
}
@end

~~~~

### 两种设置时间方式
主要注意的是,`dispatch_source_set_timer`需要一个参数`dispatch_time_t,而dispatch_time_t`的创建有两种

- dispatch_time(dispatch_time_t when, int64_t delta)

~~~~
第一个参数是从什么时间开始,一般直接传 DISPATCH_TIME_NOW; 表示从现在开始
第二个参数表示具体的时间长度(不能直接传 int 或 float), 可以写成这种形式 (int64_t)2* NSEC_PER_SEC
        
        #define NSEC_PER_SEC 1000000000ull  表示每秒有1000000000纳秒
        #define NSEC_PER_MSEC 1000000ull    表示每毫秒有1000000纳秒
        #define USEC_PER_SEC 1000000ull     表示每秒有1000000微秒
        #define NSEC_PER_USEC 1000ull       表示每微秒有1000纳秒
        
  注意 delta 的单位是纳秒! 
 1秒可以写成是 1* NSEC_PER_SEC 或者 1000* NSEC_PER_MSEC 或者 USEC_PER_SEC* NSEC_PER_USEC
~~~~

- `dispatch_time_t
dispatch_time(dispatch_time_t when, int64_t delta);
dispatch_walltime(const struct timespec *_Nullable when, int64_t delta)`

~~~~
第一个参数是一个结构体, 创建的是一个绝对的时间点,比如 2016年10月10日8点30分30秒, 如果你不需要自某一个特定的时刻开始,可以传 NUll,表示自动获取当前时区的当前时间作为开始时刻,。
第二参数意义同第一个函数

dispatch_time_t
dispatch_walltime(const struct timespec *_Nullable when, int64_t delta);
~~~~

### 这两种方式的区别是：

> 例如: 从现在开始,1小时之后是触发某个事件

> 使用第一个函数创建的是一个相对的时间,第一个参数开始时间参考的是当前系统的时钟,当 device 进入休眠之后,系统的时钟也会进入休眠状态, 第一个函数同样被挂起; 假如 device 在第一个函数开始执行后10分钟进入了休眠状态,那么这个函数同时也会停止执行,当你再次唤醒 device 之后,该函数同时被唤醒,但是事件的触发就变成了从唤醒 device 的时刻开始,1小时之后触发某个事件

> 而第二个函数则不同,他创建的是一个绝对的时间点,一旦创建就表示从这个时间点开始,1小时之后触发事件,假如 device 休眠了10分钟,当再次唤醒 device 的时候,计算时间间隔的时间起点还是 开始时就设置的那个时间点, 而不会受到 device 是否进入休眠影响

这种方式可以更加精确地使用定时器，因为是直接跟内核挂钩的，跟`RunLoop`没有关系，所以也不会有常见的`RunLoop`模式的改变而导致定时器的暂停等问题。如果我们需要对定时器的精度要求很高的话，可以考虑`dispatch_source_t`去实现

