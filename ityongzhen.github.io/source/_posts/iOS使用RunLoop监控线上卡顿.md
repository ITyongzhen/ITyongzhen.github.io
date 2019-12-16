---
layout: post
title: iOS使用RunLoop监控线上卡顿
date: 2018-05-16 15:32:24.000000000 +09:00
categories: 
- iOS
---


本文首发于[我的个人博客](https://ityongzhen.github.io/iOS%E4%BD%BF%E7%94%A8RunLoop%E7%9B%91%E6%8E%A7%E7%BA%BF%E4%B8%8A%E5%8D%A1%E9%A1%BF.html)

## 前言

关于性能优化，我之前写过[iOS性能优化](https://juejin.im/post/5d1a00846fb9a07ed657eb3f)，经过优化之后，我们的APP，冷启动，从2.7秒优化到了0.6秒。

关RunLoop，写过[RunLoop详解之源码分析](https://juejin.im/post/5d04c88d5188255e1305ca09),以及[详解RunLoop与多线程
](https://ityongzhen.github.io/%E8%AF%A6%E8%A7%A3RunLoop%E4%B8%8E%E5%A4%9A%E7%BA%BF%E7%A8%8B.html)，那么使用RunLoop如何来监控性能卡顿呢。
通过[iOS性能优化](https://juejin.im/post/5d1a00846fb9a07ed657eb3f) 我们知道，简单来说App卡顿，就是FPS达不到60帧率，丢帧现象，就会卡顿。但是很多时候，我们只知道丢帧了。具体为什么丢帧，却不是很清楚，那么我们要怎么监控呢，首先我们要明白，要找出卡顿，就是要找出主线程做了什么，而线程消息，是依赖RunLoop的，所以我们可以使用RunLoop来监控。
>RunLoop是用来监听输入源，进行调度处理的。如果RunLoop的线程进入睡眠前方法的执行时间过长而导致无法进入睡眠，或者线程唤醒后接收消息时间过长而无法进入下一步，就可以认为是线程受阻了。如果这个线程是主线程的话，表现出来的就是出现了卡顿。
>

## RunLoop和信号量

**我们可以使用CFRunLoopObserverRef来监控NSRunLoop的状态,通过它可以实时获得这些状态值的变化。**


### runloop

关于runloop,可以参照 [RunLoop详解之源码分析](https://juejin.im/post/5d04c88d5188255e1305ca09) 这篇文章详细了解。这里简单总结一下: 

- runloop的状态

````
/* Run Loop Observer Activities */
typedef CF_OPTIONS(CFOptionFlags, CFRunLoopActivity) {
    kCFRunLoopEntry = (1UL << 0),           // 即将进入Loop
    kCFRunLoopBeforeTimers = (1UL << 1),    //即将处理Timer
    kCFRunLoopBeforeSources = (1UL << 2),   //即将处理Source
    kCFRunLoopBeforeWaiting = (1UL << 5),   //即将进入休眠
    kCFRunLoopAfterWaiting = (1UL << 6),    //刚从休眠中唤醒
    kCFRunLoopExit = (1UL << 7),            //即将退出Loop
    kCFRunLoopAllActivities = 0x0FFFFFFFU   //所有状态改变
};
````
- CFRunLoopObserverRef 的使用流程
	
	1. 设置Runloop observer的运行环境

	~~~~
	CFRunLoopObserverContext context = {0, (__bridge void *)self, NULL, NULL};
	~~~~


	2. 创建Runloop observer对象
	
	~~~~
	第一个参数：用于分配observer对象的内存
	第二个参数：用以设置observer所要关注的事件
	第三个参数：用于标识该observer是在第一次进入runloop时执行还是每次进入runloop处理时均执行
	第四个参数：用于设置该observer的优先级
	第五个参数：用于设置该observer的回调函数
	第六个参数：用于设置该observer的运行环境
	 // 创建Runloop observer对象
    _observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                        kCFRunLoopAllActivities,
                                        YES,
                                        0,
                                        &runLoopObserverCallBack,
                                        &context);
	~~~~


	3. 将新建的observer加入到当前thread的runloop

	~~~~
	CFRunLoopAddObserver(CFRunLoopGetMain(), _observer, kCFRunLoopCommonModes);
	~~~~

	4. 将observer从当前thread的runloop中移除
	
	~~~~
	CFRunLoopRemoveObserver(CFRunLoopGetMain(), _observer, kCFRunLoopCommonModes);
	~~~~

	5. 释放 observer

	~~~~
	CFRelease(_observer); _observer = NULL;
	~~~~

### 信号量

关于信号量，可以详细参考 [GCD信号量-dispatch_semaphore_t](https://www.jianshu.com/p/24ffa819379c) 

简单来说,主要有三个函数

~~~~

dispatch_semaphore_create(long value); // 创建信号量
dispatch_semaphore_signal(dispatch_semaphore_t deem); // 发送信号量
dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout); // 等待信号量

~~~~

>>dispatch_semaphore_create(long value);和GCD的group等用法一致，这个函数是创建一个dispatch_semaphore_类型的信号量，并且创建的时候需要指定信号量的大小。
dispatch_semaphore_wait(dispatch_semaphore_t dsema, dispatch_time_t timeout); 等待信号量。如果信号量值为0，那么该函数就会一直等待，也就是不返回（相当于阻塞当前线程），直到该函数等待的信号量的值大于等于1，该函数会对信号量的值进行减1操作，然后返回。
dispatch_semaphore_signal(dispatch_semaphore_t deem); 发送信号量。该函数会对信号量的值进行加1操作。
通常等待信号量和发送信号量的函数是成对出现的。并发执行任务时候，在当前任务执行之前，用dispatch_semaphore_wait函数进行等待（阻塞），直到上一个任务执行完毕后且通过dispatch_semaphore_signal函数发送信号量（使信号量的值加1），dispatch_semaphore_wait函数收到信号量之后判断信号量的值大于等于1，会再对信号量的值减1，然后当前任务可以执行，执行完毕当前任务后，再通过dispatch_semaphore_signal函数发送信号量（使信号量的值加1），通知执行下一个任务......如此一来，通过信号量，就达到了并发队列中的任务同步执行的要求。


## 监控卡顿

### 原理: 利用观察Runloop各种状态变化的持续时间来检测计算是否发生卡顿
 一次有效卡顿采用了“N次卡顿超过阈值T”的判定策略，即一个时间段内卡顿的次数累计大于N时才触发采集和上报：举例，卡顿阈值T=500ms、卡顿次数N=1，可以判定为单次耗时较长的一次有效卡顿；而卡顿阈值T=50ms、卡顿次数N=5，可以判定为频次较快的一次有效卡顿

### 主要代码

~~~~
// minimum
static const NSInteger MXRMonitorRunloopMinOneStandstillMillisecond = 20;
static const NSInteger MXRMonitorRunloopMinStandstillCount = 1;

// default
// 超过多少毫秒为一次卡顿
static const NSInteger MXRMonitorRunloopOneStandstillMillisecond = 50;
// 多少次卡顿纪录为一次有效卡顿
static const NSInteger MXRMonitorRunloopStandstillCount = 1;

@interface YZMonitorRunloop(){
    CFRunLoopObserverRef _observer;  // 观察者
    dispatch_semaphore_t _semaphore; // 信号量
    CFRunLoopActivity _activity;     // 状态
}
@property (nonatomic, assign) BOOL isCancel; //f是否取消检测
@property (nonatomic, assign) NSInteger countTime; // 耗时次数
@property (nonatomic, strong) NSMutableArray *backtrace;

~~~~

~~~~
-(void)registerObserver{
//    1. 设置Runloop observer的运行环境
    CFRunLoopObserverContext context = {0, (__bridge void *)self, NULL, NULL};
    // 2. 创建Runloop observer对象
  
//    第一个参数：用于分配observer对象的内存
//    第二个参数：用以设置observer所要关注的事件
//    第三个参数：用于标识该observer是在第一次进入runloop时执行还是每次进入runloop处理时均执行
//    第四个参数：用于设置该observer的优先级
//    第五个参数：用于设置该observer的回调函数
//    第六个参数：用于设置该observer的运行环境
    _observer = CFRunLoopObserverCreate(kCFAllocatorDefault,
                                        kCFRunLoopAllActivities,
                                        YES,
                                        0,
                                        &runLoopObserverCallBack,
                                        &context);
    // 3. 将新建的observer加入到当前thread的runloop
    CFRunLoopAddObserver(CFRunLoopGetMain(), _observer, kCFRunLoopCommonModes);
    // 创建信号  dispatchSemaphore的知识参考：https://www.jianshu.com/p/24ffa819379c
    _semaphore = dispatch_semaphore_create(0); ////Dispatch Semaphore保证同步
    
    __weak __typeof(self) weakSelf = self;
    
    //    dispatch_queue_t queue = dispatch_queue_create("kadun", NULL);
    
    // 在子线程监控时长
    dispatch_async(dispatch_get_global_queue(0, 0), ^{
        //      dispatch_async(queue, ^{
        __strong __typeof(weakSelf) strongSelf = weakSelf;
        if (!strongSelf) {
            return;
        }
        while (YES) {
            if (strongSelf.isCancel) {
                return;
            }
            // N次卡顿超过阈值T记录为一次卡顿
            // 等待信号量：如果信号量是0，则阻塞当前线程；如果信号量大于0，则此函数会把信号量-1，继续执行线程。此处超时时间设为limitMillisecond 毫秒。
            // 返回值：如果线程是唤醒的，则返回非0，否则返回0
            long semaphoreWait = dispatch_semaphore_wait(self->_semaphore, dispatch_time(DISPATCH_TIME_NOW, strongSelf.limitMillisecond * NSEC_PER_MSEC));
            
            if (semaphoreWait != 0) {
                
                // 如果 RunLoop 的线程，进入睡眠前方法的执行时间过长而导致无法进入睡眠(kCFRunLoopBeforeSources)，或者线程唤醒后接收消息时间过长(kCFRunLoopAfterWaiting)而无法进入下一步的话，就可以认为是线程受阻。
                //两个runloop的状态，BeforeSources和AfterWaiting这两个状态区间时间能够监测到是否卡顿
                if (self->_activity == kCFRunLoopBeforeSources || self->_activity == kCFRunLoopAfterWaiting) {
                    
                    if (++strongSelf.countTime < strongSelf.standstillCount){
                        NSLog(@"%ld",strongSelf.countTime);
                        continue;
                    }
                    [strongSelf logStack];
                    [strongSelf printLogTrace];
                    
                    NSString *backtrace = [YZCallStack yz_backtraceOfMainThread];
                    NSLog(@"++++%@",backtrace);
                    
                    [[YZLogFile sharedInstance] writefile:backtrace];
                    
                    if (strongSelf.callbackWhenStandStill) {
                        strongSelf.callbackWhenStandStill();
                    }
                }
            }
            strongSelf.countTime = 0;
        }
    });
}
~~~~

### demo测试

我把demo放在了github [demo地址](https://github.com/ITyongzhen/YZMonitorRunloop)

使用时候，只需要 

~~~~

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.
    
    [[YZMonitorRunloop sharedInstance] startMonitor];
    [YZMonitorRunloop sharedInstance].callbackWhenStandStill = ^{
        NSLog(@"eagle.检测到卡顿了");
    };
    return YES;
}

~~~~

控制器中，每次点击屏幕，休眠1秒钟，如下

~~~~
#import "ViewController.h"

@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
}
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
     usleep(1 * 1000 * 1000); // 1秒
   
}

@end
~~~~

点击屏幕之后，打印如下 

~~~~
YZMonitorRunLoopDemo[10288:1915706] ==========检测到卡顿之后调用堆栈==========
 (
    "0   YZMonitorRunLoopDemo                0x00000001022c653c -[YZMonitorRunloop logStack] + 96",
    "1   YZMonitorRunLoopDemo                0x00000001022c62a0 __36-[YZMonitorRunloop registerObserver]_block_invoke + 484",
    "2   libdispatch.dylib                   0x00000001026ab6f0 _dispatch_call_block_and_release + 24",
    "3   libdispatch.dylib                   0x00000001026acc74 _dispatch_client_callout + 16",
    "4   libdispatch.dylib                   0x00000001026afad4 _dispatch_queue_override_invoke + 876",
    "5   libdispatch.dylib                   0x00000001026bddc8 _dispatch_root_queue_drain + 372",
    "6   libdispatch.dylib                   0x00000001026be7ac _dispatch_worker_thread2 + 156",
    "7   libsystem_pthread.dylib             0x00000001b534d1b4 _pthread_wqthread + 464",
    "8   libsystem_pthread.dylib             0x00000001b534fcd4 start_wqthread + 4"
) 

libsystem_kernel.dylib          0x1b52ca400 __semwait_signal + 8
libsystem_c.dylib               0x1b524156c nanosleep + 212
libsystem_c.dylib               0x1b5241444 usleep + 64
YZMonitorRunLoopDemo            0x1022c18dc -[ViewController touchesBegan:withEvent:] + 76
UIKitCore                       0x1e1f4fcdc <redacted> + 336
UIKitCore                       0x1e1f4fb78 <redacted> + 60
UIKitCore                       0x1e1f5e0f8 <redacted> + 1584
UIKitCore                       0x1e1f5f52c <redacted> + 3140
UIKitCore                       0x1e1f3f59c <redacted> + 340
UIKitCore                       0x1e2005714 <redacted> + 1768
UIKitCore                       0x1e2007e40 <redacted> + 4828
UIKitCore                       0x1e2001070 <redacted> + 152
CoreFoundation                  0x1b56bf018 <redacted> + 24
CoreFoundation                  0x1b56bef98 <redacted> + 88
CoreFoundation                  0x1b56be880 <redacted> + 176
CoreFoundation                  0x1b56b97
~~~~

即可定位到卡顿位置
> -[ViewController touchesBegan:withEvent:]

## 卡顿日志写入本地

上面已经监控到了卡顿，和调用堆栈。如果是debug模式下，可以直接看日志，如果想在线上查看的话，可以写入本地，然后上传到服务器

### 写入本地数据库

- 创建本地路径 
	
~~~~
	-(NSString *)getLogPath{
    NSArray *paths  = NSSearchPathForDirectoriesInDomains(NSCachesDirectory,NSUserDomainMask,YES);
    NSString *homePath = [paths objectAtIndex:0];
    
    NSString *filePath = [homePath stringByAppendingPathComponent:@"Caton.log"];
    return filePath;
}

~~~~

- 如果是第一次写入，带上设备信息，手机型号等信息

~~~~
 NSString *filePath = [self getLogPath];
    NSFileManager *fileManager = [NSFileManager defaultManager];
    
    if(![fileManager fileExistsAtPath:filePath]) //如果不存在
    {
        NSString *str = @"卡顿日志";
        NSString *systemVersion = [NSString stringWithFormat:@"手机版本: %@",[YZAppInfoUtil iphoneSystemVersion]];
        NSString *iphoneType = [NSString stringWithFormat:@"手机型号: %@",[YZAppInfoUtil iphoneType]];
        str = [NSString stringWithFormat:@"%@\n%@\n%@",str,systemVersion,iphoneType];
        [str writeToFile:filePath atomically:YES encoding:NSUTF8StringEncoding error:nil];
        
    }
~~~~

- 如果本地文件已经存在，就先判断大小是否过大，决定是否直接写入，还是先上传到服务器

~~~~
 float filesize = -1.0;
 if ([fileManager fileExistsAtPath:filePath]) {
            NSDictionary *fileDic = [fileManager attributesOfItemAtPath:filePath error:nil];
            unsigned long long size = [[fileDic objectForKey:NSFileSize] longLongValue];
            filesize = 1.0 * size / 1024;
  }
        
 NSLog(@"文件大小 filesize = %lf",filesize);
 NSLog(@"文件内容 %@",string);
 NSLog(@" ---------------------------------");
        
if (filesize > (self.MAXFileLength > 0 ? self.MAXFileLength:DefaultMAXLogFileLength)) {
     // 上传到服务器
       NSLog(@" 上传到服务器");
       [self update];
       [self clearLocalLogFile];
       [self writeToLocalLogFilePath:filePath contentStr:string];
   }else{
        NSLog(@"继续写入本地");
        [self writeToLocalLogFilePath:filePath contentStr:string];
   }
~~~~

## 压缩日志，上传服务器

因为都是文本数据，所以我们可以压缩之后，打打降低占用空间，然后进行上传，上传成功之后，删除本地，然后继续写入，等待下次写日志

### 压缩工具

使用 [SSZipArchive](https://github.com/wuhaiwei/SSZipArchive)具体使用起来也很简单，

~~~~
// Unzipping
NSString *zipPath = @"path_to_your_zip_file";
NSString *destinationPath = @"path_to_the_folder_where_you_want_it_unzipped";
[SSZipArchive unzipFileAtPath:zipPath toDestination:destinationPath];
// Zipping
NSString *zippedPath = @"path_where_you_want_the_file_created";
NSArray *inputPaths = [NSArray arrayWithObjects:
                       [[NSBundle mainBundle] pathForResource:@"photo1" ofType:@"jpg"],
                       [[NSBundle mainBundle] pathForResource:@"photo2" ofType:@"jpg"]
                       nil];
[SSZipArchive createZipFileAtPath:zippedPath withFilesAtPaths:inputPaths];
~~~~


代码中

~~~~
  NSString *zipPath = [self getLogZipPath];
    NSString *password = nil;
    NSMutableArray *filePaths = [[NSMutableArray alloc] init];
    [filePaths addObject:[self getLogPath]];
    BOOL success = [SSZipArchive createZipFileAtPath:zipPath withFilesAtPaths:filePaths withPassword:password.length > 0 ? password : nil];
    
    if (success) {
        NSLog(@"压缩成功");
        
    }else{
        NSLog(@"压缩失败");
    }
~~~~

具体如果上传到服务器，使用者可以用AFN等将本地的 zip文件上传到文件服务器即可，就不赘述了。

至此，我们做到了，用runloop，监控卡顿，写入日志，然后压缩上传服务器，删除本地的过程。

详细代码见[demo地址](https://github.com/ITyongzhen/YZMonitorRunloop)

参考资料 ：

[BSBacktraceLogger](https://github.com/bestswifter/BSBacktraceLogger)

[GCD信号量-dispatch_semaphore_t](https://www.jianshu.com/p/24ffa819379c)

[SSZipArchive](https://github.com/wuhaiwei/SSZipArchive)

[简单监测iOS卡顿的demo](https://www.jianshu.com/p/71cfbcb15842)

[RunLoop实战：实时卡顿监控](https://juejin.im/post/5cacb2baf265da03904bf93b)


