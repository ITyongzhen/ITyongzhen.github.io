---
layout: post
title: 关于iOS中的13种加锁方案
date: 2017-11-15 15:32:24.000000000 +09:00
categories: 
- iOS
---

本文首发于[个人博客](https://ityongzhen.github.io/%E5%85%B3%E4%BA%8EiOS%E4%B8%AD%E7%9A%8413%E7%A7%8D%E5%8A%A0%E9%94%81%E6%96%B9%E6%A1%88.html)


## 前言

iOS中有很多锁，那么平时使用过程中到底怎么使用呢？本文分享13种加锁方案。本文较长总共一万字。文中代码在[github](https://github.com/ITyongzhen/iOS-ManyLocks)上。


- `OSSpinLock`自旋锁
- `os_unfair_lock`互斥锁
- `pthread_mutex`递归锁
- `pthread_mutex`条件锁
- `dispatch_semaphore`信号量
- `dispatch_queue(DISPATCH_QUEUE_SERIAL)`
- `NSLock`
- `NSRecursiveLock`
- `NSCondition`
- `NSConditionLock`
- `@synchronized`
- `dispatch_barrier_async`栅栏
- `dispatch_group`调度组

性能对比：借用ibireme大神的一张图片 

![](https://user-gold-cdn.xitu.io/2019/8/15/16c9511ccc9cfbb0?w=568&h=359&f=png&s=34948)

可以看到除了 `OSSpinLock` 外，`dispatch_semaphore` 和 `pthread_mutex` 性能是最高的。现在苹果在新系统中已经优化了 `pthread_mutex `的性能，所以它看上去和 `OSSpinLock` 差距并没有那么大了



GNUstep是GNU计划的项目之一，它将Cocoa的OC库重新开源实现了一遍

[GNUstep源码地址](http://www.gnustep.org/resources/downloads.php)

虽然[GNUstep](http://www.gnustep.org/resources/downloads.php)不是苹果官方源码，但还是具有一定的参考价值


### 自旋锁

wikipedia中关于[自旋锁](https://zh.wikipedia.org/wiki/%E8%87%AA%E6%97%8B%E9%94%81)的描述

> 自旋锁是计算机科学用于多线程同步的一种锁，线程反复检查锁变量是否可用。由于线程在这一过程中保持执行，因此是一种忙等待。一旦获取了自旋锁，线程会一直保持该锁，直至显式释放自旋锁。

> 自旋锁避免了进程上下文的调度开销，因此对于线程只会阻塞很短时间的场合是有效的。因此操作系统的实现在很多地方往往用自旋锁。Windows操作系统提供的轻型读写锁（SRW Lock）内部就用了自旋锁。显然，单核CPU不适于使用自旋锁，这里的单核CPU指的是单核单线程的CPU，因为，在同一时间只有一个线程是处在运行状态，假设运行线程A发现无法获取锁，只能等待解锁，但因为A自身不挂起，所以那个持有锁的线程B没有办法进入运行状态，只能等到操作系统分给A的时间片用完，才能有机会被调度。这种情况下使用自旋锁的代价很高。

### 互斥锁

wikipedia中关于[互斥锁](https://zh.wikipedia.org/wiki/%E4%BA%92%E6%96%A5%E9%94%81)的描述

>互斥锁（英语：Mutual exclusion，缩写 Mutex）是一种用于多线程编程中，防止两条线程同时对同一公共资源（比如全局变量）进行读写的机制。该目的通过将代码切片成一个一个的临界区域（critical section）达成。临界区域指的是一块对公共资源进行访问的代码，并非一种机制或是算法。一个程序、进程、线程可以拥有多个临界区域，但是并不一定会应用互斥锁。

需要此机制的资源的例子有：旗标、队列、计数器、中断处理程序等用于在多条并行运行的代码间传递数据、同步状态等的资源。维护这些资源的同步、一致和完整是很困难的，因为一条线程可能在任何一个时刻被暂停（休眠）或者恢复（唤醒）。

例如：一段代码（甲）正在分步修改一块数据。这时，另一条线程（乙）由于一些原因被唤醒。如果乙此时去读取甲正在修改的数据，而甲碰巧还没有完成整个修改过程，这个时候这块数据的状态就处在极大的不确定状态中，读取到的数据当然也是有问题的。更严重的情况是乙也往这块地方写数据，这样的一来，后果将变得不可收拾。因此，多个线程间共享的数据必须被保护。达到这个目的的方法，就是确保同一时间只有一个临界区域处于运行状态，而其他的临界区域，无论是读是写，都必须被挂起并且不能获得运行机会。

### 读写锁

wikipedia中关于[互斥锁](https://zh.wikipedia.org/wiki/%E4%BA%92%E6%96%A5%E9%94%81)的描述

>读写锁是计算机程序的并发控制的一种同步机制，也称“共享-互斥锁”、多读者-单写者锁。多读者锁，“push lock”) 用于解决读写问题。读操作可并发重入，写操作是互斥的。

>读写锁通常用互斥锁、条件变量、信号量实现。

读写锁可以有不同的操作模式优先级：

- 读操作优先：允许最大并发，但写操作可能饿死。
- 写操作优先：一旦所有已经开始的读操作完成，等待的写操作立即获得锁。内部实现需要两把互斥锁。
- 未指定优先级

### 信号量

wikipedia中关于[信号量](https://zh.wikipedia.org/wiki/%E4%BF%A1%E5%8F%B7%E9%87%8F)的描述



>信号量（英语：semaphore）又称为信号标，是一个同步对象，用于保持在0至指定最大值之间的一个计数值。当线程完成一次对该semaphore对象的等待（wait）时，该计数值减一；当线程完成一次对semaphore对象的释放（release）时，计数值加一。当计数值为0，则线程等待该semaphore对象不再能成功直至该semaphore对象变成signaled状态。semaphore对象的计数值大于0，为signaled状态；计数值等于0，为nonsignaled状态.

>semaphore对象适用于控制一个仅支持有限个用户的共享资源，是一种不需要使用忙碌等待（busy waiting）的方法。

>信号量的概念是由荷兰计算机科学家艾兹赫尔·戴克斯特拉（Edsger W. Dijkstra）发明的，广泛的应用于不同的操作系统中。在系统中，给予每一个进程一个信号量，代表每个进程当前的状态，未得到控制权的进程会在特定地方被强迫停下来，等待可以继续进行的信号到来。如果信号量是一个任意的整数，通常被称为计数信号量（Counting semaphore），或一般信号量（general semaphore）；如果信号量只有二进制的0或1，称为二进制信号量（binary semaphore）。在linux系统中，二进制信号量（binary semaphore）又称互斥锁（Mutex）



## 场景

使用经典的存钱-取钱案例。假设我们账号里面有100元，每次存钱都存10元，每次取钱都取20元。存5次，取5次。那么就是应该最终剩下50元才对。如果我们把存在和取钱在不同的线程中访问的时候，如果不加锁，就很可能导致问题。

~~~~
/**
 存钱、取钱演示
 */
- (void)moneyTest
{
    self.money = 100;
    
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    dispatch_async(queue, ^{
        for (int i = 0; i < 5; i++) {
            [self __saveMoney];
        }
    });
    
    dispatch_async(queue, ^{
        for (int i = 0; i < 5; i++) {
            [self __drawMoney];
        }
    });
}

/**
 存钱
 */
- (void)__saveMoney
{
    int oldMoney = self.money;
    sleep(.2);
    oldMoney += 10;
    self.money = oldMoney;
    
    NSLog(@"存10元，还剩%d元 - %@", oldMoney, [NSThread currentThread]);
}

/**
 取钱
 */
- (void)__drawMoney
{
    int oldMoney = self.money;
    sleep(.2);
    oldMoney -= 20;
    self.money = oldMoney;
    
    NSLog(@"取20元，还剩%d元 - %@", oldMoney, [NSThread currentThread]);
}

~~~~

输出结果为

~~~~
iOS-LockDemo[21005:249636] 存10元，还剩110元 - <NSThread: 0x600001b4f540>{number = 4, name = (null)}
iOS-LockDemo[21005:249637] 取20元，还剩80元 - <NSThread: 0x600001b79840>{number = 3, name = (null)}
iOS-LockDemo[21005:249636] 存10元，还剩120元 - <NSThread: 0x600001b4f540>{number = 4, name = (null)}
iOS-LockDemo[21005:249636] 存10元，还剩110元 - <NSThread: 0x600001b4f540>{number = 4, name = (null)}
iOS-LockDemo[21005:249637] 取20元，还剩100元 - <NSThread: 0x600001b79840>{number = 3, name = (null)}
iOS-LockDemo[21005:249637] 取20元，还剩90元 - <NSThread: 0x600001b79840>{number = 3, name = (null)}
iOS-LockDemo[21005:249636] 存10元，还剩100元 - <NSThread: 0x600001b4f540>{number = 4, name = (null)}
iOS-LockDemo[21005:249637] 取20元，还剩80元 - <NSThread: 0x600001b79840>{number = 3, name = (null)}
iOS-LockDemo[21005:249636] 存10元，还剩90元 - <NSThread: 0x600001b4f540>{number = 4, name = (null)}
iOS-LockDemo[21005:249637] 取20元，还剩70元 - <NSThread: 0x600001b79840>{number = 3, name = (null)}
~~~~

从结果上来看，明显不是预期的那样

这是因为，正常情况下，来存钱取消，存10元之后，还剩下110元，然后取钱20元，剩余90元没问题。但是我们是不同线程同时操作的时候，可能导致的情况是，正在存钱的是，来取钱了。也就是10元还没存进去，就去取钱。取钱之后先去获取当前的钱数，因为10元正在存呢，还没存完，取钱的时候，当前是100元，然后取出20元的过程中，刚才的10元存进去了，然后20元也取出来了。给出结果是100-20 = 80 元，然后实际上应该 100+10-20 = 90 元。这样的话，就导致了数据的紊乱。

### 如何解决：

解决这种问题，就需要线程锁了。当存钱的时候，先去加锁，然后存完了，再放开锁。取钱也是一样，这样就保证数据的一致性。


## `OSSpinLock`自旋锁


- `OSSpinLock `叫做”自旋锁”，等待锁的线程会处于忙等（busy-wait）状态，一直占用着CPU资源
- 目前已经不再安全，可能会出现优先级反转问题
- 需要导入头文件`#import <libkern/OSAtomic.h>`

### 使用

~~~~
// 初始化
OSSpinLock lock = OS_SPINLOCK_INIT;
//尝试加锁(如果不需要等待，就直接加锁，返回true。如果需要等待，就不加锁，返回false)
BOOL res = OSSpinLockTry(lock);
//加锁
OSSpinLockLock(lock);
//解锁
OSSpinLockUnlock(lock);
~~~~

`YZOSSpinLock `继承`YZBaseLock`，在每次存钱，取钱之前进行加锁，在每次存钱，取钱之后进行解锁。

~~~~
#import "YZOSSpinLock.h"
#import <libkern/OSAtomic.h>

@interface YZOSSpinLock()
@property (assign, nonatomic) OSSpinLock moneyLock;
@end

@implementation YZOSSpinLock

- (instancetype)init
{
    if (self = [super init]) {
        self.moneyLock = OS_SPINLOCK_INIT;
    }
    return self;
}

- (void)__drawMoney
{
    OSSpinLockLock(&_moneyLock);
    
    [super __drawMoney];
    
    OSSpinLockUnlock(&_moneyLock);
}

- (void)__saveMoney
{
    OSSpinLockLock(&_moneyLock);
    
    [super __saveMoney];
    
    OSSpinLockUnlock(&_moneyLock);
}


@end
~~~~

输出结果为

~~~~
iOS-LockDemo[22496:265962] 取20元，还剩80元 - <NSThread: 0x600003add800>{number = 3, name = (null)}
iOS-LockDemo[22496:265962] 取20元，还剩60元 - <NSThread: 0x600003add800>{number = 3, name = (null)}
iOS-LockDemo[22496:265962] 取20元，还剩40元 - <NSThread: 0x600003add800>{number = 3, name = (null)}
iOS-LockDemo[22496:265962] 取20元，还剩20元 - <NSThread: 0x600003add800>{number = 3, name = (null)}
iOS-LockDemo[22496:265962] 取20元，还剩0元 - <NSThread: 0x600003add800>{number = 3, name = (null)}
iOS-LockDemo[22496:265961] 存10元，还剩10元 - <NSThread: 0x600003aecd00>{number = 4, name = (null)}
iOS-LockDemo[22496:265961] 存10元，还剩20元 - <NSThread: 0x600003aecd00>{number = 4, name = (null)}
iOS-LockDemo[22496:265961] 存10元，还剩30元 - <NSThread: 0x600003aecd00>{number = 4, name = (null)}
iOS-LockDemo[22496:265961] 存10元，还剩40元 - <NSThread: 0x600003aecd00>{number = 4, name = (null)}
iOS-LockDemo[22496:265961] 存10元，还剩50元 - <NSThread: 0x600003aecd00>{number = 4, name = (null)}
~~~~

由输出可知，能保证线程安全，数据没有错乱。但是`OSSpinLock`已经不再安全了。

### 汇编跟踪

在加锁的地方打断点，第二次进来的是，已经加锁了，这时候看加锁的汇编代码
>Debug->Debug Worlflow->Always Show Disassembly


### 为什么`OSSpinLock`不再安全

关于为什么`OSSpinLock`不再安全可以参考这篇文章[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

这里摘要主要内容

>如果一个低优先级的线程获得锁并访问共享资源，这时一个高优先级的线程也尝试获得这个锁，它会处于 spin lock 的忙等状态从而占用大量 CPU。此时低优先级线程无法与高优先级线程争夺 CPU 时间，从而导致任务迟迟完不成、无法释放 lock，这就是优先级反转。这并不只是理论上的问题，开发者已经遇到很多次这个问题，于是苹果工程师停用了 OSSpinLock。


### 结论

- 除非开发者能保证访问锁的线程全部都处于同一优先级，否则 iOS 系统中所有类型的自旋锁都不能再使用了。


## `os_unfair_lock `互斥锁

`os_unfair_lock`用于取代不安全的OSSpinLock ，从iOS10开始才支持
从底层调用看，等待`os_unfair_lock`锁的线程会处于休眠状态，并非忙等

需要导入头文件`#import <os/lock.h>`

### 使用

~~~~
// 初始化
os_unfair_lock lock = OS_UNFAIR_LOCK_INIT;
//尝试加锁(如果不需要等待，就直接加锁，返回true。如果需要等待，就不加锁，返回false)
BOOL res = os_unfair_lock_trylock(&lock);
//加锁
os_unfair_lock_lock(&lock);
//解锁
os_unfair_lock_unlock(&lock);
~~~~

`YZUnfairLock `继承`YZBaseLock`，在每次存钱，取钱之前进行加锁，在每次存钱，取钱之后进行解锁。

~~~~

#import "YZUnfairLock.h"
#import <os/lock.h>

@interface YZUnfairLock()
@property (nonatomic ,assign) os_unfair_lock moneyLock;

@end

@implementation YZUnfairLock
- (instancetype)init
{
    
    if (self = [super init]) {
        self.moneyLock = OS_UNFAIR_LOCK_INIT;
    }
    return self;
}

- (void)__saveMoney
{
    os_unfair_lock_lock(&_moneyLock);
    
    [super __saveMoney];
    
    os_unfair_lock_unlock(&_moneyLock);
}

- (void)__drawMoney
{
    os_unfair_lock_lock(&_moneyLock);
    
    [super __drawMoney];
    
    os_unfair_lock_unlock(&_moneyLock);
}

@end
~~~~

### 汇编跟踪

在加锁的地方打断点，第二次进来的是，已经加锁了，这时候看加锁的汇编代码
>Debug->Debug Worlflow->Always Show Disassembly

断点跟踪进去，会发现最终到`syscall`的时候，断点失效了。这是因为syscall调用了系统内核的函数，使得线程进入休眠状态，不再占用CPU资源。所以可以看出`os_unfair_lock `是互斥锁。

![](https://user-gold-cdn.xitu.io/2019/8/15/16c9511ccd641e81?w=889&h=245&f=png&s=68205)


## `pthread_mutex`互斥锁


- mutex叫做”互斥锁”，等待锁的线程会处于休眠状态
- 需要导入头文件#import <pthread.h>

### 使用

~~~~
// 初始化属性
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_DEFAULT);
// 初始化锁
pthread_mutex_init(mutex, &attr);
// 销毁属性
pthread_mutexattr_destroy(&attr);
~~~~

其中锁的类型有四种

~~~~
#define PTHREAD_MUTEX_NORMAL		0   //一般的锁
#define PTHREAD_MUTEX_ERRORCHECK	1	// 错误检查
#define PTHREAD_MUTEX_RECURSIVE		2  //递归锁
#define PTHREAD_MUTEX_DEFAULT		PTHREAD_MUTEX_NORMAL  //默认
~~~~

当类型是`PTHREAD_MUTEX_DEFAULT `的时候，相当于`null`

例如上面的使用可以直接等价于

~~~~
 pthread_mutex_init(mutex, NULL); //传空，相当于PTHREAD_MUTEX_DEFAULT
~~~~

`YZMutexLock `继承`YZBaseLock`，在每次存钱，取钱之前进行加锁，在每次存钱，取钱之后进行解锁。

~~~~
#import "YZMutexLock.h"
#import <pthread.h>

@interface YZMutexLock()
@property (assign, nonatomic) pthread_mutex_t moneyMutexLock;

@end

@implementation YZMutexLock

/**
 初始化锁

 @param mutex 锁
 */
- (void)__initMutexLock:(pthread_mutex_t *)mutex{
    // 静态初始化
    //            pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
    
    // 初始化属性
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_DEFAULT);
    // 初始化锁
    pthread_mutex_init(mutex, &attr);
    // 销毁属性
    pthread_mutexattr_destroy(&attr);
    
	// 上面五行相当于下面一行
    //pthread_mutex_init(mutex, NULL); //传空，相当于PTHREAD_MUTEX_DEFAULT
}


- (instancetype)init
{
    self = [super init];
    if (self) {
        [self __initMutexLock:&_moneyMutexLock];
    }
    return self;
}
- (void)__saveMoney
{
    pthread_mutex_lock(&_moneyMutexLock);
    
    [super __saveMoney];
    
     pthread_mutex_unlock(&_moneyMutexLock);
}

- (void)__drawMoney
{
     pthread_mutex_lock(&_moneyMutexLock);
    
    [super __drawMoney];
    
     pthread_mutex_unlock(&_moneyMutexLock);
}
- (void)dealloc
{
    //delloc时候，需要销毁锁
    pthread_mutex_destroy(&_moneyMutexLock);
}
@end
~~~~

看到输出也是没问题的。线程是安全的。

~~~~
iOS-LockDemo[2573:45093] 存10元，还剩110元 - <NSThread: 0x600003ebbb80>{number = 3, name = (null)}
iOS-LockDemo[2573:45093] 存10元，还剩120元 - <NSThread: 0x600003ebbb80>{number = 3, name = (null)}
iOS-LockDemo[2573:45093] 存10元，还剩130元 - <NSThread: 0x600003ebbb80>{number = 3, name = (null)}
iOS-LockDemo[2573:45095] 取20元，还剩110元 - <NSThread: 0x600003e84880>{number = 4, name = (null)}
iOS-LockDemo[2573:45095] 取20元，还剩90元 - <NSThread: 0x600003e84880>{number = 4, name = (null)}
iOS-LockDemo[2573:45095] 取20元，还剩70元 - <NSThread: 0x600003e84880>{number = 4, name = (null)}
iOS-LockDemo[2573:45095] 取20元，还剩50元 - <NSThread: 0x600003e84880>{number = 4, name = (null)}
iOS-LockDemo[2573:45095] 取20元，还剩30元 - <NSThread: 0x600003e84880>{number = 4, name = (null)}
iOS-LockDemo[2573:45093] 存10元，还剩40元 - <NSThread: 0x600003ebbb80>{number = 3, name = (null)}
iOS-LockDemo[2573:45093] 存10元，还剩50元 - <NSThread: 0x600003ebbb80>{number = 3, name = (null)}

~~~~

## `pthread_mutex`递归锁


- mutex除了有”互斥锁”，还有递归锁
- 需要导入头文件#import <pthread.h>

### 使用

~~~~
// 初始化属性
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
// 初始化锁
pthread_mutex_init(mutex, &attr);
// 销毁属性
pthread_mutexattr_destroy(&attr);
~~~~

其中锁的类型有四种

~~~~
#define PTHREAD_MUTEX_NORMAL		0   //一般的锁
#define PTHREAD_MUTEX_ERRORCHECK	1	// 错误检查
#define PTHREAD_MUTEX_RECURSIVE		2  //递归锁
#define PTHREAD_MUTEX_DEFAULT		PTHREAD_MUTEX_NORMAL  //默认
~~~~

eg:

`YZMutexRecursiveLock `继承`YZBaseLock`，`otherTest`里面进行递归加锁

~~~~
#import "YZMutexRecursiveLock.h"
#import <pthread.h>

@interface YZMutexRecursiveLock()
@property (assign, nonatomic) pthread_mutex_t MutexLock;

@end

@implementation YZMutexRecursiveLock

/**
 初始化锁
 
 @param mutex 锁
 */
- (void)__initMutexLock:(pthread_mutex_t *)mutex{
    // 递归锁：允许同一个线程对一把锁进行重复加锁
    
    // 初始化属性
    pthread_mutexattr_t attr;
    pthread_mutexattr_init(&attr);
    pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
    // 初始化锁
    pthread_mutex_init(mutex, &attr);
    // 销毁属性
    pthread_mutexattr_destroy(&attr);

    
}

- (void)otherTest{
    // 第一次进来直接加锁，第二次进来，已经加锁了。还能递归继续加锁
    pthread_mutex_lock(&_MutexLock);
    NSLog(@"加锁 %s",__func__);
    static int count = 0;
    if (count < 5) {
        count++;
        [self otherTest];
    }
     NSLog(@"解锁 %s",__func__);
    pthread_mutex_unlock(&_MutexLock);
    
}
- (instancetype)init
{
    self = [super init];
    if (self) {
        [self __initMutexLock:&_MutexLock];
    }
    return self;
}

- (void)dealloc
{
    //delloc时候，需要销毁锁
    pthread_mutex_destroy(&_MutexLock);
}
@end
~~~~


调用的时候

~~~~
YZBaseLock *lock = [[YZMutexRecursiveLock alloc] init];
[lock otherTest];
~~~~

输出结果为：

~~~~
iOS-LockDemo[7358:129676] 加锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 加锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 加锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 加锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 加锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 加锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 解锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 解锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 解锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 解锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 解锁 -[YZMutexRecursiveLock otherTest]
iOS-LockDemo[7358:129676] 解锁 -[YZMutexRecursiveLock otherTest]
~~~~

由结果可知，连续加锁五次，是因为每次都递归加锁。然后解锁时候，层层解锁。


## `pthread_mutex`条件锁

![](https://user-gold-cdn.xitu.io/2019/8/15/16c9511ccd6f9cc3?w=1034&h=690&f=png&s=348934)

- mutex除了有"互斥锁"，"递归锁"，还有递归锁
- 需要导入头文件#import <pthread.h>

### 使用

#### 生产者消费者

为了演示条件锁的作用，就用生产者消费者来展示效果，关于生产者消费者的设计模式，可以看我之前的文章[iOS设计模式之(二)生产者-消费者](https://juejin.im/post/5d27dce4e51d4510774a88f0)，那篇文章中用的是信号量实现的。这篇文章用`pthread_mutex`条件锁来实现。

#### 代码

有三个属性

~~~~
@property (assign, nonatomic) pthread_mutex_t mutex; // 锁
@property (assign, nonatomic) pthread_cond_t cond; //条件
@property (strong, nonatomic) NSMutableArray *data; //数据源

~~~~

初始化

~~~~
// 初始化属性
pthread_mutexattr_t attr;
pthread_mutexattr_init(&attr);
pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
// 初始化锁
pthread_mutex_init(&_mutex, &attr);
// 销毁属性
pthread_mutexattr_destroy(&attr);
        
// 初始化条件
pthread_cond_init(&_cond, NULL);
        
self.data = [NSMutableArray array];
~~~~



eg:

`YZMutexCondLock `继承`YZBaseLock`，`otherTest`里面进行测试

~~~~
//
//  YZMutexCondLock.m
//  iOS-LockDemo
//
//  Created by eagle on 2018/8/13.
//  Copyright © 2018 yongzhen. All rights reserved.
//

#import "YZMutexCondLock.h"
#import <pthread.h>

@interface YZMutexCondLock()
@property (assign, nonatomic) pthread_mutex_t mutex; // 锁
@property (assign, nonatomic) pthread_cond_t cond; //条件
@property (strong, nonatomic) NSMutableArray *data; //数据源
@end

@implementation YZMutexCondLock
- (instancetype)init
{
    if (self = [super init]) {
        // 初始化属性
        pthread_mutexattr_t attr;
        pthread_mutexattr_init(&attr);
        pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_RECURSIVE);
        // 初始化锁
        pthread_mutex_init(&_mutex, &attr);
        // 销毁属性
        pthread_mutexattr_destroy(&attr);
        
        // 初始化条件
        pthread_cond_init(&_cond, NULL);
        
        self.data = [NSMutableArray array];
    }
    return self;
}

- (void)otherTest
{
    [[[NSThread alloc] initWithTarget:self selector:@selector(__remove) object:nil] start];
    
    [[[NSThread alloc] initWithTarget:self selector:@selector(__add) object:nil] start];
}


// 生产者-消费者模式

// 线程1
// 删除数组中的元素
- (void)__remove
{
    pthread_mutex_lock(&_mutex);
    
    
    if (self.data.count == 0) {
        // 数据为空就等待（进入休眠，放开mutex锁，被唤醒后，会再次对mutex加锁）
        NSLog(@"__remove - 等待");
        pthread_cond_wait(&_cond, &_mutex);
    }
    
    [self.data removeLastObject];
    NSLog(@"删除了元素");
    
    pthread_mutex_unlock(&_mutex);
}

// 线程2
// 往数组中添加元素
- (void)__add
{
    pthread_mutex_lock(&_mutex);
    
    sleep(1);
    
    [self.data addObject:@"Test"];
    NSLog(@"添加了元素");
    
    // 激活一个等待该条件的线程
    pthread_cond_signal(&_cond);
    // 激活所有等待该条件的线程
    //    pthread_cond_broadcast(&_cond);
    
    pthread_mutex_unlock(&_mutex);
}

- (void)dealloc
{
    pthread_mutex_destroy(&_mutex);
    pthread_cond_destroy(&_cond);
}


@end

~~~~


调用的时候

~~~~
YZBaseLock *lock = [[YZMutexCondLock alloc] init];
[lock otherTest];
~~~~

输出结果为：

~~~~
2018-08-13 17:09:31.643902+0800 iOS-LockDemo[26733:229374] __remove - 等待
2018-08-13 17:09:32.648587+0800 iOS-LockDemo[26733:229375] 添加了元素
2018-08-13 17:09:32.648894+0800 iOS-LockDemo[26733:229374] 删除了元素
~~~~

由结果可知，打印完`__remove - 等待`之后，等待了一秒钟，添加元素之后，放开锁，才去删除元素。

## NSLock锁

- `NSLock`是对`mutex`普通锁的封装

### api


`NSLocking `协议有加锁`lock `和解锁`unlock `，

~~~~
@protocol NSLocking

- (void)lock;
- (void)unlock;

@end
~~~~

NSLock准守这个协议，锁可以直接使用，另外，还有`tryLock`和`lockBeforeDate `

~~~~
- (void)lock; //加锁
- (void)unlock; //解锁
- (BOOL)tryLock; //尝试加锁，如果加锁失败，就返回NO,加锁成功就返回YES
- (BOOL)lockBeforeDate:(NSDate *)limit; //在给定的时间内尝试加锁，加锁成功就返回YES,如果过了时间还没加上锁，就返回NO。
~~~~

### 使用


`YZNSLock `继承`YZBaseLock`，在每次存钱，取钱之前进行加锁，在每次存钱，取钱之后进行解锁。

~~~~
#import "YZNSLock.h"

@interface YZNSLock()
@property (nonatomic,strong) NSLock *lock;
@end


@implementation YZNSLock

- (instancetype)init
{
    self = [super init];
    if (self) {
        self.lock =[[NSLock alloc] init];
    }
    return self;
}
- (void)__saveMoney
{
    [self.lock lock];
    [super __saveMoney];
    
    [self.lock unlock];
}

- (void)__drawMoney
{
    [self.lock lock];
    
    [super __drawMoney];
    
    [self.lock unlock];
}
- (void)dealloc
{
   
}


@end
~~~~

输出结果为

~~~~
iOS-LockDemo[39175:397286] 存10元，还剩110元 - <NSThread: 0x600000af2740>{number = 3, name = (null)}
iOS-LockDemo[39175:397286] 存10元，还剩120元 - <NSThread: 0x600000af2740>{number = 3, name = (null)}
iOS-LockDemo[39175:397286] 存10元，还剩130元 - <NSThread: 0x600000af2740>{number = 3, name = (null)}
iOS-LockDemo[39175:397287] 取20元，还剩110元 - <NSThread: 0x600000ae66c0>{number = 4, name = (null)}
iOS-LockDemo[39175:397286] 存10元，还剩120元 - <NSThread: 0x600000af2740>{number = 3, name = (null)}
iOS-LockDemo[39175:397287] 取20元，还剩100元 - <NSThread: 0x600000ae66c0>{number = 4, name = (null)}
iOS-LockDemo[39175:397287] 取20元，还剩80元 - <NSThread: 0x600000ae66c0>{number = 4, name = (null)}
iOS-LockDemo[39175:397287] 取20元，还剩60元 - <NSThread: 0x600000ae66c0>{number = 4, name = (null)}
iOS-LockDemo[39175:397287] 取20元，还剩40元 - <NSThread: 0x600000ae66c0>{number = 4, name = (null)}
iOS-LockDemo[39175:397286] 存10元，还剩50元 - <NSThread: 0x600000af2740>{number = 3, name = (null)}
~~~~

由输出可知，能保证线程安全，数据没有错乱。

### `NSLock`是对`mutex`普通锁的封装

如果想证明`NSLock`是对`mutex`普通锁的封装有两种方式

- 汇编分析
	- 汇编分析来说，可以打断点跟进去，最终会发现调用了`mutex `，因为，lock是调用的msgSend,汇编代码比较复杂，读者有兴趣可自行验证。
- [GNUstep](http://www.gnustep.org/resources/downloads.php)
	- [GNUstep](http://www.gnustep.org/resources/downloads.php)源码的NSLock.m中如下代码

	
~~~~
+ (void) initialize
{
  static BOOL	beenHere = NO;

  if (beenHere == NO)
    {
      beenHere = YES;
      pthread_mutexattr_init(&attr_normal);
      pthread_mutexattr_settype(&attr_normal, PTHREAD_MUTEX_NORMAL);
      pthread_mutexattr_init(&attr_reporting);
      pthread_mutexattr_settype(&attr_reporting, PTHREAD_MUTEX_ERRORCHECK);
      pthread_mutexattr_init(&attr_recursive);
      pthread_mutexattr_settype(&attr_recursive, PTHREAD_MUTEX_RECURSIVE);
      
      ... 其他代码
}
~~~~


## NSRecursiveLock锁

- NSRecursiveLock也是对mutex递归锁的封装，API跟NSLock基本一致

### api


`NSLocking `协议有加锁`lock `和解锁`unlock `，

~~~~
@protocol NSLocking

- (void)lock;
- (void)unlock;

@end
~~~~

NSRecursiveLock准守这个协议，可以直接使用，另外，还有`tryLock`和`lockBeforeDate `

~~~~
- (void)lock; //加锁
- (void)unlock; //解锁
- (BOOL)tryLock; //尝试加锁，如果加锁失败，就返回NO,加锁成功就返回YES
- (BOOL)lockBeforeDate:(NSDate *)limit; //在给定的时间内尝试加锁，加锁成功就返回YES,如果过了时间还没加上锁，就返回NO。
~~~~

### 使用


`YZNSRecursiveLock `继承`YZBaseLock`，在每次存钱，取钱之前进行加锁，在每次存钱，取钱之后进行解锁。

~~~~
#import "YZNSRecursiveLock.h"

@interface YZNSRecursiveLock()
@property (nonatomic,strong) NSRecursiveLock *lock;
@end


@implementation YZNSRecursiveLock

- (instancetype)init
{
    self = [super init];
    if (self) {
        self.lock =[[NSRecursiveLock alloc] init];
    }
    return self;
}
- (void)__saveMoney
{
    [self.lock lock];
    [super __saveMoney];
    
    [self.lock unlock];
}

- (void)__drawMoney
{
    [self.lock lock];
    
    [super __drawMoney];
    
    [self.lock unlock];
}
- (void)dealloc
{
    
}
@end
~~~~

输出结果为

~~~~
iOS-LockDemo[39175:397286] 存10元，还剩110元 - <NSThread: 0x600000af2740>{number = 3, name = (null)}
iOS-LockDemo[39175:397286] 存10元，还剩120元 - <NSThread: 0x600000af2740>{number = 3, name = (null)}
iOS-LockDemo[39175:397286] 存10元，还剩130元 - <NSThread: 0x600000af2740>{number = 3, name = (null)}
iOS-LockDemo[39175:397287] 取20元，还剩110元 - <NSThread: 0x600000ae66c0>{number = 4, name = (null)}
iOS-LockDemo[39175:397286] 存10元，还剩120元 - <NSThread: 0x600000af2740>{number = 3, name = (null)}
iOS-LockDemo[39175:397287] 取20元，还剩100元 - <NSThread: 0x600000ae66c0>{number = 4, name = (null)}
iOS-LockDemo[39175:397287] 取20元，还剩80元 - <NSThread: 0x600000ae66c0>{number = 4, name = (null)}
iOS-LockDemo[39175:397287] 取20元，还剩60元 - <NSThread: 0x600000ae66c0>{number = 4, name = (null)}
iOS-LockDemo[39175:397287] 取20元，还剩40元 - <NSThread: 0x600000ae66c0>{number = 4, name = (null)}
iOS-LockDemo[39175:397286] 存10元，还剩50元 - <NSThread: 0x600000af2740>{number = 3, name = (null)}
~~~~

由输出可知，能保证线程安全，数据没有错乱。

### `YZNSRecursiveLock `是对`mutex`递归锁的封装

如果想证明`NSRecursiveLock `是对`mutex`普通锁的封装有两种方式

- 汇编分析
	- 汇编分析来说，可以打断点跟进去，最终会发现调用了`mutex `，因为，lock是调用的msgSend,汇编代码比较复杂，读者有兴趣可自行验证。
- [GNUstep](http://www.gnustep.org/resources/downloads.php)
	- [GNUstep](http://www.gnustep.org/resources/downloads.php)源码的NSLock.m中的NSRecursiveLock有如下代码

	
~~~~
//NSRecursiveLock初始化
- (id) init
{
  if (nil != (self = [super init]))
    {
      if (0 != pthread_mutex_init(&_mutex, &attr_recursive))
	{
	  DESTROY(self);
	}
    }
  return self;
}

// attr_recursive初始化
pthread_mutexattr_init(&attr_recursive);
pthread_mutexattr_settype(&attr_recursive, PTHREAD_MUTEX_RECURSIVE);
      
~~~~

 
## `NSCondition `条件锁

- NSCondition是对mutex和cond的封装


### 使用

#### 生产者消费者

同上面的`YZMutexCondLock`一样使用生产者消费者模式



#### api
~~~~
- (void)wait; //等待
- (BOOL)waitUntilDate:(NSDate *)limit; //在给定时间之前等待
- (void)signal;   // 激活一个等待该条件的线程
- (void)broadcast;  // 激活所有等待该条件的线程

~~~~

#### 代码

初始化

~~~~
// 初始化属性
 self.condition = [[NSCondition alloc] init];
~~~~



eg:

`YZNSCondition `继承`YZBaseLock`，`otherTest`里面进行测试

~~~~
#import "YZNSCondition.h"

@interface YZNSCondition()
@property (strong, nonatomic) NSCondition *condition;
@property (strong, nonatomic) NSMutableArray *data;
@end

@implementation YZNSCondition

- (instancetype)init
{
    if (self = [super init]) {
        self.condition = [[NSCondition alloc] init];
        self.data = [NSMutableArray array];
    }
    return self;
}

- (void)otherTest
{
    [[[NSThread alloc] initWithTarget:self selector:@selector(__remove) object:nil] start];
    
    [[[NSThread alloc] initWithTarget:self selector:@selector(__add) object:nil] start];
}
// 生产者-消费者模式

// 线程1
// 删除数组中的元素
- (void)__remove
{
    [self.condition lock];
    
    
    if (self.data.count == 0) {
        // 数据为空就等待（进入休眠，放开锁，被唤醒后，会再次对mutex加锁）
        NSLog(@"__remove - 等待");
        [self.condition wait];
    }
    
    [self.data removeLastObject];
    NSLog(@"删除了元素");
    
    [self.condition unlock];
}

// 线程2
// 往数组中添加元素
- (void)__add
{
    [self.condition lock];
    
    sleep(1);
    
    [self.data addObject:@"Test"];
    NSLog(@"添加了元素");
    
    // 激活一个等待该条件的线程
    [self.condition signal];
    // 激活所有等待该条件的线程
    //     [self.condition broadcast];
    
    [self.condition unlock];
}

@end

~~~~


调用的时候

~~~~
YZBaseLock *lock = [[YZNSCondition alloc] init];
[lock otherTest];
~~~~

输出结果为：

~~~~
2018-08-13 18:09:31.643902+0800 iOS-LockDemo[26733:229374] __remove - 等待
2018-08-13 18:09:32.648587+0800 iOS-LockDemo[26733:229375] 添加了元素
2018-08-13 18:09:32.648894+0800 iOS-LockDemo[26733:229374] 删除了元素
~~~~

由结果可知，打印完`__remove - 等待`之后，等待了一秒钟，添加元素之后，放开锁，才去删除元素。

## NSConditionLock

NSConditionLock是对NSCondition的进一步封装，可以设置具体的条件值



### API

主要有如下几个API，顾名思义，一看名字就懂了。

~~~~
- (instancetype)initWithCondition:(NSInteger)condition NS_DESIGNATED_INITIALIZER;

@property (readonly) NSInteger condition;
- (void)lockWhenCondition:(NSInteger)condition;
- (BOOL)tryLock;
- (BOOL)tryLockWhenCondition:(NSInteger)condition;
- (void)unlockWithCondition:(NSInteger)condition;
- (BOOL)lockBeforeDate:(NSDate *)limit;
- (BOOL)lockWhenCondition:(NSInteger)condition beforeDate:(NSDate *)limit;

~~~~

### 使用

初始化

~~~~
// 初始化属性
NSConditionLock *lock = [[NSConditionLock alloc] initWithCondition:0];

//当条件为2的时候，加锁
[lock lockWhenCondition:2];

//当条件为3的时候，解锁
 [lock unlockWithCondition:3];

~~~~


eg:

`YZNSConditionLock `继承`YZBaseLock`，`otherTest`里面进行测试

~~~~
#import "YZNSConditionLock.h"

@interface YZNSConditionLock()
@end

@implementation YZNSConditionLock

- (void)otherTest
{
    //主线程中
    NSConditionLock *lock = [[NSConditionLock alloc] initWithCondition:0];
    
    //线程1
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [lock lockWhenCondition:2];
        NSLog(@"线程1");
        sleep(2);
        NSLog(@"线程1解锁成功");
        [lock unlockWithCondition:3];
    });
    
    //线程2
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [lock lockWhenCondition:0];
        NSLog(@"线程2");
        sleep(3);
        NSLog(@"线程2解锁成功");
        [lock unlockWithCondition:1];
    });
    
    //线程3
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [lock lockWhenCondition:3];
        NSLog(@"线程3");
        sleep(3);
        NSLog(@"线程3解锁成功");
        [lock unlockWithCondition:4];
    });
    
    //线程4
    dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0), ^{
        [lock lockWhenCondition:1];
        NSLog(@"线程4");
        sleep(2);
        NSLog(@"线程4解锁成功");
        [lock unlockWithCondition:2];
    });
    
}
@end

~~~~


调用的时候

~~~~
YZBaseLock *lock = [[YZNSConditionLock alloc] init];
[lock otherTest];
~~~~

输出结果为：

~~~~
2018-08-14 15:37:07.850783+0800 iOS-LockDemo[11810:143479] 线程2
2018-08-14 15:37:10.854390+0800 iOS-LockDemo[11810:143479] 线程2解锁成功
2018-08-14 15:37:10.854703+0800 iOS-LockDemo[11810:143478] 线程4
2018-08-14 15:37:12.856226+0800 iOS-LockDemo[11810:143478] 线程4解锁成功
2018-08-14 15:37:12.856487+0800 iOS-LockDemo[11810:143476] 线程1
2018-08-14 15:37:14.860596+0800 iOS-LockDemo[11810:143476] 线程1解锁成功
2018-08-14 15:37:14.860791+0800 iOS-LockDemo[11810:143477] 线程3
2018-08-14 15:37:17.864072+0800 iOS-LockDemo[11810:143477] 线程3解锁成功
~~~~

由结果可知，NSConditionLock完全能够通过条件值进行加锁解锁。

## `dispatch_semaphore`信号量

- semaphore叫做”信号量”
- 信号量的初始值，可以用来控制线程并发访问的最大数量
- 信号量的初始值为1，代表同时只允许1条线程访问资源，保证线程同步

关于信号量详细可以参考[`GCD信号量-dispatch_semaphore_t`](https://www.jianshu.com/p/24ffa819379c) 以及对信号量实际应用,结合RunLoop做成卡顿监控的[iOS使用RunLoop监控线上卡顿](https://ityongzhen.github.io/iOS%E4%BD%BF%E7%94%A8RunLoop%E7%9B%91%E6%8E%A7%E7%BA%BF%E4%B8%8A%E5%8D%A1%E9%A1%BF.html)


### 信号量原理

~~~~
dispatch_semaphore_create(long value); // 创建信号量
dispatch_semaphore_signal(dispatch_semaphore_t deem); // 发送信号量
dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout); // 等待信号量

~~~~

>`dispatch_semaphore_create(long value)`和GCD的group等用法一致，这个函数是创建一个`dispatch_semaphore_`类型的信号量，并且创建的时候需要指定信号量的大小。

>`dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout)` 等待信号量。如果信号量值为0，那么该函数就会一直等待，也就是不返回（相当于阻塞当前线程），直到该函数等待的信号量的值大于等于1，该函数会对信号量的值进行减1操作，然后返回。

>`dispatch_semaphore_signal(dispatch_semaphore_t deem)` 发送信号量。该函数会对信号量的值进行加1操作。

>通常等待信号量和发送信号量的函数是成对出现的。并发执行任务时候，在当前任务执行之前，用`dispatch_semaphore_wait`函数进行等待（阻塞），直到上一个任务执行完毕后且通过`dispatch_semaphore_signal`函数发送信号量（使信号量的值加1），`dispatch_semaphore_wait`函数收到信号量之后判断信号量的值大于等于1，会再对信号量的值减1，然后当前任务可以执行，执行完毕当前任务后，再通过`dispatch_semaphore_signal`函数发送信号量（使信号量的值加1），通知执行下一个任务......如此一来，通过信号量，就达到了并发队列中的任务同步执行的要求。

### 使用

先看加锁，解锁的使用，初始化先设置1，然后每次取钱，存钱之前，都调用`dispatch_semaphore_wait `,取钱，存钱之后调用`dispatch_semaphore_signal `

eg:

`YZSemaphore `继承`YZBaseLock`，`otherTest`里面进行测试

~~~~
#import "YZSemaphore.h"

@interface YZSemaphore()
@property (strong, nonatomic) dispatch_semaphore_t moneySemaphore;
@end

@implementation YZSemaphore
- (instancetype)init
{
    if (self = [super init]) {
        self.moneySemaphore = dispatch_semaphore_create(1);
    }
    return self;
}

- (void)__drawMoney
{
    dispatch_semaphore_wait(self.moneySemaphore, DISPATCH_TIME_FOREVER);
    
    [super __drawMoney];
    
    dispatch_semaphore_signal(self.moneySemaphore);
}

- (void)__saveMoney
{
    dispatch_semaphore_wait(self.moneySemaphore, DISPATCH_TIME_FOREVER);
    
    [super __saveMoney];
    
    dispatch_semaphore_signal(self.moneySemaphore);
}
~~~~

外部调用的时候，

~~~~
YZBaseLock *lock = [[YZSemaphore alloc] init];
[lock moneyTest];
~~~~

输出

~~~~
iOS-LockDemo[13500:171371] 存10元，还剩110元 - <NSThread: 0x600001ca9840>{number = 3, name = (null)}
iOS-LockDemo[13500:171369] 取20元，还剩90元 - <NSThread: 0x600001c960c0>{number = 4, name = (null)}
iOS-LockDemo[13500:171371] 存10元，还剩100元 - <NSThread: 0x600001ca9840>{number = 3, name = (null)}
iOS-LockDemo[13500:171369] 取20元，还剩80元 - <NSThread: 0x600001c960c0>{number = 4, name = (null)}
iOS-LockDemo[13500:171371] 存10元，还剩90元 - <NSThread: 0x600001ca9840>{number = 3, name = (null)}
iOS-LockDemo[13500:171369] 取20元，还剩70元 - <NSThread: 0x600001c960c0>{number = 4, name = (null)}
iOS-LockDemo[13500:171371] 存10元，还剩80元 - <NSThread: 0x600001ca9840>{number = 3, name = (null)}
iOS-LockDemo[13500:171369] 取20元，还剩60元 - <NSThread: 0x600001c960c0>{number = 4, name = (null)}
iOS-LockDemo[13500:171371] 存10元，还剩70元 - <NSThread: 0x600001ca9840>{number = 3, name = (null)}
iOS-LockDemo[13500:171369] 取20元，还剩50元 - <NSThread: 0x600001c960c0>{number = 4, name = (null)}
~~~~

有结果可知，能保证多线程数据的安全读写。

### 使用二

信号量还可以控制线程数量，例如初始化的时候，设置最多3条线程

~~~~
#import "YZSemaphore.h"

@interface YZSemaphore()
@property (strong, nonatomic) dispatch_semaphore_t semaphore;
@end

@implementation YZSemaphore
- (instancetype)init
{
    if (self = [super init]) {
        self.semaphore = dispatch_semaphore_create(3);
    }
    return self;
}

- (void)otherTest
{
    for (int i = 0; i < 20; i++) {
        [[[NSThread alloc] initWithTarget:self selector:@selector(test) object:nil] start];
    }
}

// 线程10、7、6、9、8
- (void)test
{
    // 如果信号量的值 > 0，就让信号量的值减1，然后继续往下执行代码
    // 如果信号量的值 <= 0，就会休眠等待，直到信号量的值变成>0，就让信号量的值减1，然后继续往下执行代码
    dispatch_semaphore_wait(self.semaphore, DISPATCH_TIME_FOREVER);
    
    sleep(2);
    NSLog(@"test - %@", [NSThread currentThread]);
    
    // 让信号量的值+1
    dispatch_semaphore_signal(self.semaphore);
}

@end
~~~~

调用`otherTest `的输出结果为

~~~~
2018-08-14 16:38:56.489121+0800 iOS-LockDemo[14002:180654] test - <NSThread: 0x600003a938c0>{number = 3, name = (null)}
2018-08-14 16:38:56.492100+0800 iOS-LockDemo[14002:180655] test - <NSThread: 0x600003a93900>{number = 4, name = (null)}
2018-08-14 16:38:56.492281+0800 iOS-LockDemo[14002:180656] test - <NSThread: 0x600003a93940>{number = 5, name = (null)}

2018-08-14 16:38:58.497578+0800 iOS-LockDemo[14002:180657] test - <NSThread: 0x600003a93980>{number = 6, name = (null)}
2018-08-14 16:38:58.499225+0800 iOS-LockDemo[14002:180658] test - <NSThread: 0x600003a8e640>{number = 7, name = (null)}
2018-08-14 16:38:58.549633+0800 iOS-LockDemo[14002:180659] test - <NSThread: 0x600003a93a00>{number = 8, name = (null)}

2018-08-14 16:39:00.499672+0800 iOS-LockDemo[14002:180660] test - <NSThread: 0x600003aa6cc0>{number = 9, name = (null)}
2018-08-14 16:39:00.499799+0800 iOS-LockDemo[14002:180661] test - <NSThread: 0x600003aa6ec0>{number = 10, name = (null)}
2018-08-14 16:39:00.550353+0800 iOS-LockDemo[14002:180662] test - <NSThread: 0x600003aa6d80>{number = 11, name = (null)}

2018-08-14 16:39:02.501379+0800 iOS-LockDemo[14002:180663] test - <NSThread: 0x600003aa6f00>{number = 12, name = (null)}
~~~~

由结果可知，每次最多三条线程执行。

## synchronized

- @synchronized是对mutex递归锁的封装
- 源码查看：objc4中的objc-sync.mm文件
- @synchronized(obj)内部会生成obj对应的递归锁，然后进行加锁、解锁操作


详细了解可以参考 [关于 @synchronized，这儿比你想知道的还要多](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)

### 使用

@synchronized 使用起来很简单

还使用前面的存钱取票的例子，类YZSynchronized继承自YZBaseLock，代码如下

~~~~
#import "YZSynchronized.h"

@interface YZSynchronized()

@end

@implementation YZSynchronized
- (void)__saveMoney
{
    @synchronized (self) {
        [super __saveMoney];
    }
}

- (void)__drawMoney
{
    @synchronized (self) {
        [super __drawMoney];
    }
}
@end
~~~~

调用之后

~~~~
iOS-LockDemo[2573:45093] 存10元，还剩110元 - <NSThread: 0x600003ebbb80>{number = 3, name = (null)}
iOS-LockDemo[2573:45093] 存10元，还剩120元 - <NSThread: 0x600003ebbb80>{number = 3, name = (null)}
iOS-LockDemo[2573:45093] 存10元，还剩130元 - <NSThread: 0x600003ebbb80>{number = 3, name = (null)}
iOS-LockDemo[2573:45095] 取20元，还剩110元 - <NSThread: 0x600003e84880>{number = 4, name = (null)}
iOS-LockDemo[2573:45095] 取20元，还剩90元 - <NSThread: 0x600003e84880>{number = 4, name = (null)}
iOS-LockDemo[2573:45095] 取20元，还剩70元 - <NSThread: 0x600003e84880>{number = 4, name = (null)}
iOS-LockDemo[2573:45095] 取20元，还剩50元 - <NSThread: 0x600003e84880>{number = 4, name = (null)}
iOS-LockDemo[2573:45095] 取20元，还剩30元 - <NSThread: 0x600003e84880>{number = 4, name = (null)}
iOS-LockDemo[2573:45093] 存10元，还剩40元 - <NSThread: 0x600003ebbb80>{number = 3, name = (null)}
iOS-LockDemo[2573:45093] 存10元，还剩50元 - <NSThread: 0x600003ebbb80>{number = 3, name = (null)}

~~~~

可知，多线程的数据没有发生错乱

### 源码分析

从[runtime源码](https://opensource.apple.com/tarballs/objc4/)中的objc-sync.mm中可知

~~~~
int objc_sync_enter(id obj)
{
    int result = OBJC_SYNC_SUCCESS;

    if (obj) {
        SyncData* data = id2data(obj, ACQUIRE);
        assert(data);
        data->mutex.lock();
    } else {
        // @synchronized(nil) does nothing
        if (DebugNilSync) {
            _objc_inform("NIL SYNC DEBUG: @synchronized(nil); set a breakpoint on objc_sync_nil to debug");
        }
        objc_sync_nil();
    }

    return result;
}


// End synchronizing on 'obj'. 
// Returns OBJC_SYNC_SUCCESS or OBJC_SYNC_NOT_OWNING_THREAD_ERROR
int objc_sync_exit(id obj)
{
    int result = OBJC_SYNC_SUCCESS;
    
    if (obj) {
        SyncData* data = id2data(obj, RELEASE); 
        if (!data) {
            result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
        } else {
            bool okay = data->mutex.tryUnlock();
            if (!okay) {
                result = OBJC_SYNC_NOT_OWNING_THREAD_ERROR;
            }
        }
    } else {
        // @synchronized(nil) does nothing
    }
	

    return result;
}
~~~~

~~~~
typedef struct alignas(CacheLineSize) SyncData {
    struct SyncData* nextData;
    DisguisedPtr<objc_object> object;
    int32_t threadCount;  // number of THREADS using this block
    recursive_mutex_t mutex;
} SyncData;
~~~~

以及

~~~~
using recursive_mutex_t = recursive_mutex_tt<LOCKDEBUG>;
~~~~

可知@synchronized是对mutex递归锁的封装。因为是递归锁，可以递归加锁，读者有兴趣自行验证。


## `pthread_rwlock`读写锁

>读写锁是计算机程序的并发控制的一种同步机制，也称“共享-互斥锁”、多读者-单写者锁。多读者锁，“push lock”) 用于解决读写问题。读操作可并发重入，写操作是互斥的。

- 需要导入头文件#import <pthread.h>


#### 使用

~~~~
// 初始化锁
pthread_rwlock_t lock;
pthread_rwlock_init(&lock, NULL);
// 读-加锁
pthread_rwlock_rdlock(&lock);
// 读-尝试加锁
pthread_rwlock_tryrdlock(&lock);
// 写-加锁
pthread_rwlock_wrlock(&lock);
// 写-尝试加锁
pthread_rwlock_trywrlock(&lock);
// 解锁
pthread_rwlock_unlock(&lock);
// 销毁
pthread_rwlock_destroy(&lock);
~~~~



eg:

`YZRwlock `继承`YZBaseLock`，`otherTest`里面进行测试，每次读，或者写的之前进行加锁，并sleep 1秒钟，之后解锁，如下所示

~~~~
#import "YZRwlock.h"
#import <pthread.h>

@interface YZRwlock()
@property (assign, nonatomic) pthread_rwlock_t lock;

@end


@implementation YZRwlock

- (instancetype)init
{
    self = [super init];
    if (self) {
        // 初始化锁
        pthread_rwlock_init(&_lock, NULL);
    }
    return self;
}

- (void)otherTest{
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    for (int i = 0; i < 3; i++) {
        dispatch_async(queue, ^{
            [self write];
            [self read];
        });
        
    }
    for (int i = 0; i < 3; i++) {
        dispatch_async(queue, ^{
            [self write];
        });
    }
}

- (void)read {
    pthread_rwlock_rdlock(&_lock);
    
    sleep(1);
    NSLog(@"%s", __func__);
    
    pthread_rwlock_unlock(&_lock);
}

- (void)write
{
    pthread_rwlock_wrlock(&_lock);
    
    sleep(1);
    NSLog(@"%s", __func__);
    
    pthread_rwlock_unlock(&_lock);
}

- (void)dealloc
{
    pthread_rwlock_destroy(&_lock);
}

@end
~~~~


调用的时候

~~~~
YZBaseLock *lock = [[YZRwlock alloc] init];
[lock otherTest];
~~~~

输出结果为：

~~~~
2018-08-15 16:07:45.753659+0800 iOS-LockDemo[25457:248359] -[YZRwlock write]
2018-08-15 16:07:46.758460+0800 iOS-LockDemo[25457:248356] -[YZRwlock write]
2018-08-15 16:07:47.763705+0800 iOS-LockDemo[25457:248358] -[YZRwlock write]
2018-08-15 16:07:48.767980+0800 iOS-LockDemo[25457:248381] -[YZRwlock write]
2018-08-15 16:07:49.772241+0800 iOS-LockDemo[25457:248382] -[YZRwlock write]
2018-08-15 16:07:50.777547+0800 iOS-LockDemo[25457:248383] -[YZRwlock write]
2018-08-15 16:07:51.779544+0800 iOS-LockDemo[25457:248359] -[YZRwlock read]
2018-08-15 16:07:51.779544+0800 iOS-LockDemo[25457:248356] -[YZRwlock read]
2018-08-15 16:07:51.779546+0800 iOS-LockDemo[25457:248358] -[YZRwlock read]
~~~~

由结果可知，打印完`write`之后，方法每次都是一个一个执行的，而`read `是可以同时执行的，也就是说达到了多读单写的功能。被称为读写锁。



## `dispatch_barrier_async`异步栅栏


- 这个函数传入的并发队列必须是自己通过`dispatch_queue_cretate`创建的
- 如果传入的是一个串行或是一个全局的并发队列，那这个函数便等同于`dispatch_async`函数的效果



### 使用

~~~~
// 初始化队列
self.queue = dispatch_queue_create("rw_queue", DISPATCH_QUEUE_CONCURRENT);

// 读
dispatch_async(self.queue, ^{
       
});
        
// 写
dispatch_barrier_async(self.queue, ^{
       
});

~~~~



eg:

`YZBarrier `继承`YZBaseLock`，`otherTest`里面进行测试

~~~~
#import "YZBarrier.h"

@interface YZBarrier ()
@property (strong, nonatomic) dispatch_queue_t queue;
@end
@implementation YZBarrier

- (void)otherTest{
    
    // 初始化队列
    self.queue = dispatch_queue_create("rw_queue", DISPATCH_QUEUE_CONCURRENT);
    
    for (int i = 0; i < 3; i++) {
        // 读
        dispatch_async(self.queue, ^{
            [self read];
        });
         // 写
        dispatch_barrier_async(self.queue, ^{
            [self write];
        });
         // 读
        dispatch_async(self.queue, ^{
            [self read];
        });
         // 读
        dispatch_async(self.queue, ^{
            [self read];
        });
        
    }
}

- (void)read {
    sleep(1);
    NSLog(@"read");
}

- (void)write
{
    sleep(1);
    NSLog(@"write");
}
@end
~~~~


调用的时候

~~~~
YZBaseLock *lock = [[YZBarrier alloc] init];
[lock otherTest];
~~~~

输出结果为：

~~~~
2018-08-15 17:50:45.867990+0800 iOS-LockDemo[30046:324146] read
2018-08-15 17:50:46.871969+0800 iOS-LockDemo[30046:324146] write
2018-08-15 17:50:47.876419+0800 iOS-LockDemo[30046:324146] read
2018-08-15 17:50:47.876419+0800 iOS-LockDemo[30046:324148] read
2018-08-15 17:50:47.876450+0800 iOS-LockDemo[30046:324145] read
2018-08-15 17:50:48.880739+0800 iOS-LockDemo[30046:324145] write
2018-08-15 17:50:49.885434+0800 iOS-LockDemo[30046:324145] read
2018-08-15 17:50:49.885435+0800 iOS-LockDemo[30046:324146] read
2018-08-15 17:50:49.885442+0800 iOS-LockDemo[30046:324148] read
2018-08-15 17:50:50.889361+0800 iOS-LockDemo[30046:324148] write
2018-08-15 17:50:51.894104+0800 iOS-LockDemo[30046:324148] read
2018-08-15 17:50:51.894104+0800 iOS-LockDemo[30046:324146] read
~~~~

由结果可知，打印完`write`之后，方法每次都是一个一个执行的，而`read `是可以同时执行的，但是遇到写的操作，就会把其他读或者写都会暂停，也就是说起到了栅栏的作用。


## `dispatch_group_t`调度组


前面说了这么多关于锁的使用，其实调度组也能达到类似栅栏的效果。



### api

~~~~
//1.创建调度组
dispatch_group_t group = dispatch_group_create();
//2.队列
dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
//3.调度组监听队列 标记开始本次执行
dispatch_group_enter(group);
//标记本次请求完成
dispatch_group_leave(group);
          
//4,调度组都完成了
dispatch_group_notify(group, dispatch_get_main_queue(), ^{
//执行刷新UI等操作
});

~~~~



eg:

`YZDispatchGroup `继承`YZBaseLock`，`otherTest`里面进行测试,假设的场景是，需要在子线程下载两个图片,sleep()模拟耗时操作，都下载完成之后，回到主线程刷新UI.

~~~~
#import "YZDispatchGroup.h"

@implementation YZDispatchGroup
- (instancetype)init
{
    self = [super init];
    if (self) {
        
    }
    return self;
}

- (void)otherTest{
    
    //1.创建调度组
    dispatch_group_t group = dispatch_group_create();
    //2.队列
    dispatch_queue_t queue = dispatch_get_global_queue(0, 0);
    
    //3.调度组监听队列 标记开始本次执行
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
       [self downLoadImage1];
        //标记本次请求完成
          dispatch_group_leave(group);
    });
    

    //3.调度组监听队列 标记开始本次执行
    dispatch_group_enter(group);
    dispatch_async(queue, ^{
        [self downLoadImage2];
        //标记本次请求完成
        dispatch_group_leave(group);
    });
    
    //4,调度组都完成了
    dispatch_group_notify(group, dispatch_get_main_queue(), ^{
        //执行完test1和test2之后，在进行请求test3
         [self reloadUI];
    });

}
- (void)downLoadImage1 {
    sleep(1);
    NSLog(@"%s--%@",__func__,[NSThread currentThread]);
}
- (void)downLoadImage2 {
     sleep(2);
    NSLog(@"%s--%@",__func__,[NSThread currentThread]);
}
- (void)reloadUI
{
    NSLog(@"%s--%@",__func__,[NSThread currentThread]);
}
@end
~~~~


调用的时候

~~~~
YZBaseLock *lock = [[YZBarrier alloc] init];
[lock otherTest];
~~~~

输出结果为：

~~~~
2018-08-15 19:08:35.651955+0800 iOS-LockDemo[3353:49583] -[YZDispatchGroup downLoadImage1]--<NSThread: 0x6000033ed380>{number = 3, name = (null)}
2018-08-15 19:08:36.648922+0800 iOS-LockDemo[3353:49584] -[YZDispatchGroup downLoadImage2]--<NSThread: 0x6000033e0000>{number = 4, name = (null)}
2018-08-15 19:08:36.649179+0800 iOS-LockDemo[3353:49521] -[YZDispatchGroup reloadUI]--<NSThread: 0x6000033865c0>{number = 1, name = main}

~~~~

由结果可知，子线程耗时操作，现在图片时候，主线程刷新UI不执行的，等两个图片都下载完成，才回到主线程刷新UI.

### `dispatch_group`有两个需要注意的地方

- dispatch_group_enter必须在dispatch_group_leave之前出现
- dispatch_group_enter和dispatch_group_leave必须成对出现


## 自旋锁，互斥锁的选择

前面这么多锁，那么到底平时开发中怎么选择呢？其实主要参考如下标准来选择。

### 什么情况使用自旋锁比较划算？
- 预计线程等待锁的时间很短
- 加锁的代码（临界区）经常被调用，但竞争情况很少发生
- CPU资源不紧张
- 多核处理器

### 什么情况使用互斥锁比较划算？
- 预计线程等待锁的时间较长
- 单核处理器
- 临界区有IO操作
- 临界区代码复杂或者循环量大
- 临界区竞争非常激烈




## 参考资料
[本文资料下载github](https://github.com/ITyongzhen/iOS-ManyLocks)


[OSSpinLock Is Unsafe](https://mjtsai.com/blog/2015/12/16/osspinlock-is-unsafe/)

[不再安全的 OSSpinLock](https://blog.ibireme.com/2016/01/16/spinlock_is_unsafe_in_ios/)

[GNUstep源码地址](http://www.gnustep.org/resources/downloads.php)

[Runtime源码](https://opensource.apple.com/tarballs/objc4/)

[iOS底层原理](https://ke.qq.com/course/package/11609)

[关于 @synchronized，这儿比你想知道的还要多](http://yulingtianxia.com/blog/2015/11/01/More-than-you-want-to-know-about-synchronized/)
