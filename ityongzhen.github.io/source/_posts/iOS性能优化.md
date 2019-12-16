---
layout: post
title: iOS性能优化
date: 2018-04-16 15:32:24.000000000 +09:00
categories: 
- iOS
---



本文首发于[个人博客](https://ityongzhen.github.io/%E6%80%A7%E8%83%BD%E4%BC%98%E5%8C%96.html
)

### 一、App启动优化

#### 1.App的启动可以分为2种

+ 冷启动(Cold Launch):从零开始启动APP
+ 热启动(Warm Launch):APP已经在内存中，在后台存活着，再次点击图标启动APP
  - APP启动时间的优化，主要是针对冷启动进行优化
  -  通过添加环境变量可以打印出APP的启动时间分析(Edit scheme -> Run -> Arguments) DYLD_PRINT_STATISTICS设置为1
  - 如果需要更详细的信息，那就将DYLD_PRINT_STATISTICS_DETAILS设置为1

#### 2.App 冷启动分为四大阶段

* dyld 加载可执行文件，动态库(递归加载)
* runtime
*  main() 函数执行后
* 首屏渲染完成后

##### 2.1关于dyld

+ ![image.png](https://user-gold-cdn.xitu.io/2019/7/1/16bad920214bad91?w=635&h=304&f=png&s=97261)

+ 在Mac 、iOS中，是使用了/usr/lib/dyld程序来加载动态库 

+ dynamic link editor，动态链接编辑器

+ dynamic loader，动态加载器

+ dyld 的源码 https://opensource.apple.com/tarballs/dyld/

  + ~~~~
    initializeMainExecutable 方法开始的.dyld会优先初始化动态库，然后初始化App的可执行文件。
    ~~~~


  ![image.png](https://user-gold-cdn.xitu.io/2019/7/1/16bad920213f9cce?w=1177&h=600&f=png&s=135868)



  用MachOView (<https://github.com/gdbinit/MachOView>)查看加载过程如上图

  （	备注1: 如果设置了 DYLD_PRINT_LIBRARIES，或者选中run/diagnostics 下面的 dynamic library loads 那么 dyld将会打印出什么库被加载了

​	备注2：DYLD_PRINT_STATISTICS_DETAILS  打印启动时间

​	备注3：dyly还可以抽取苹果原生库   方法： 1： launch-cache/dsc_extractor.cpp文件中 把#if(0) 以及之前的都删除，#endif也删除  2：编译clang++ -o dsc_extractor dsc_extractor.cpp 生成可执行文件 3：./dsc_extractor dyld_shared_cache_armv7s armv7s 进行抽取 ）

##### 2.2 runtime  

​	源码： https://opensource.apple.com/source/objc4/   源码分析可参考：https://www.jianshu.com/p/3019605a4fc9

​	启动APP时，runtime所做的事情有

* 调用map_images进行可执行文件内容的解析和处理
* 在load_images中调用call_load_methods，调用所有Class和Category的+load方法  进行各种objc结构的初始化(注册Objc类 、初始化类对象等等)
*  调用C++静态初始化器和__attribute__((constructor))修饰的函数
*  到此为止，可执行文件和动态库中所有的符号(Class，Protocol，Selector，IMP，...)都已经按格式成功加载到内存中，被 runtime 所管理

关于`load`和`initialize` 可参考[iOS中load和initialize](https://juejin.im/post/5d204a9af265da1b6e65c431)一文详细分析
 ##### 2.3main函数执行后

​	main() 函数执行后的阶段，指的是从 main() 函数执行开始，到 appDelegate 的 didFinishLaunchingWithOptions 方法里首屏渲染相关方法执行完成。

* 首屏初始化所需配置文件的读写操作
* 首屏列表大数据的读取
* 首屏渲染的大量计算等

###### 总结：

APP的启动由dyld主导，将可执行文件加载到内存，顺便加载所有依赖的动态库, 并由runtime负责加载成objc定义的结构,所有初始化工作结束后，dyld就会调用main函数, 接下来就是UIApplicationMain函数，AppDelegate的application:didFinishLaunchingWithOptions:方法

##### 3.App启动优化

​	按照不同的阶段

 *	dyld
    *	 减少动态库、合并一些动态库(定期清理不必要的动态库)。减少动态库加载。每个库本身都有依赖关系，苹果公司建议使用更少的动态库，苹果最多支持6个非系统的动态库合并为一个。
    *	 减少Objc类、分类的数量、减少Selector数量(定期清理不必要的类、分类) 
    *	减少C++虚函数数量， 减少C++全局变量的数量
    *	Swift尽量使用struct
*	runtime
  *	用+initialize方法和dispatch_once取代所有的__attribute__((constructor))、C++静态构造器、ObjC的+load，因为在一个+load()方法里，运行时进行方法替换操作会带来4毫秒的损耗。
*	main() 函数执行后
  * 功能级别的优化：main()函数开始执行后到首屏渲染完成前，只处理首屏相关的业务，其他的非首屏业务的初始化，监听注册，配置文件读取放在首屏渲染完成后去做
  * ReactiveCocoa创建一个信号6毫秒，+load()执行一次，4毫秒
  * 检测App耗时
    * 抓取主线程的方法调用堆栈，计算一段时间各个方法的耗时，Xcode自带的Time Profiler
    * 对objc_msgSend方法进行hook来掌握所有方法的执行耗时 objc_msgSend源码 https://opensource.apple.com/source/objc4/objc4-723/runtime/Messengers.subproj/
      * fackbook开源了fishhook的代码https://github.com/facebook/fishhook 其大致思路为：通过重新绑定符号，实现对c方法的hook。dyld是通过更新Mach-O二进制的_DATA segment特定的部分中的指针来绑定lazy和non-lazy符号，通过确认传递给rebind_symbol里每个符号更新的位置，就可以找出替换来重新绑定这些符号。
*	在不影响用户体验的前提下，尽可能将一些操作延迟，不要全部都放在finishLaunching方法中  按需加载
*	不使用xib，直接视用代码加载首页视图
*	NSUserDefaults实际上是在Library文件夹下会生产一个plist文件，如果文件太大的话一次能读取到内存中可能很耗时，这个影响需要评估，如果耗时很大的话需要拆分(需考虑老版本覆盖安装兼容问题)
*	每次用NSLog方式打印会隐式的创建一个Calendar，因此需要删减启动时各业务方打的log，或者仅仅针对内测版输出log
*	梳理应用启动时发送的所有网络请求，是否可以统一在异步线程请求

 ### 二、安装包瘦身

#### 1、安装包(IPA)主要由可执行文件、资源组成

* 资源(图片、音频、视频等)
*  采取无损压缩
*  去除没有用到的资源: https://github.com/tinymind/LSUnusedResources

#### 2、 可执行文件瘦身 

##### 2.1 编译器优化

* Strip Linked Product、Make Strings Read-Only、Symbols Hidden by Default设置为YES

* 去掉异常支持，Enable C++ Exceptions、Enable Objective-C Exceptions设置为NO， Other C Flags添加-fno-exceptions

##### 2.2利用AppCode

(https://www.jetbrains.com/objc/)检测未使用的代码:菜单栏 -> Code -> Inspect Code 

##### 2. 3编写LLVM插件检测出重复代码、未被调用的代码

##### 2.4 生成LinkMap文件，可以查看可执行文件的具体组成

![image.png](https://user-gold-cdn.xitu.io/2019/7/1/16bad920214ee40a?w=968&h=196&f=png&s=27708)

##### 2.5 可借助第三方工具解析LinkMap文件: https://github.com/huanxsd/LinkMap



### 三、卡顿问题

#### 3.1、CPU 和GPU

* CPU (Central Processing Unit，中央处理器)

  对象的创建和销毁、对象属性的调整、布局计算、文本的计算和排版、图片的格式转换和解码、图像的绘制(Core Graphics)

* GPU (Graphics Processing Unit，图形处理器)

  纹理的渲染

* 在iOS中是双缓冲机制，有前帧缓存、后帧缓存

#### 3.2优化方向

* 尽可能减少CPU、GPU资源消耗 
* 尽量用轻量级的对象，比如用不到事件处理的地方，可以考虑使用CALayer取代UIView
*  不要频繁地调用UIView的相关属性，比如frame、bounds、transform等属性，尽量减少不必要的修改 
*  尽量提前计算好布局，在有需要时一次性调整对应的属性，不要多次修改属性
*  Autolayout会比直接设置frame消耗更多的CPU资源
*  图片的size最好刚好跟UIImageView的size保持一致
* 控制一下线程的最大并发数量
*  尽量避免短时间内大量图片的显示，尽可能将多张图片合成一张进行显示
*  GPU能处理的最大纹理尺寸是4096x4096，一旦超过这个尺寸，就会占用CPU资源进行处理，所以纹理尽量不要超过这个尺寸
*  尽量减少视图数量和层次
* 减少透明的视图(alpha<1)，不透明的就设置opaque为YES 
* 尽量把耗时的操作放到子线程 
  -  文本处理(尺寸计算、绘制) p
  -  图片处理(解码、绘制)

#### 3.3、离屏渲染

*  尽量避免出现离屏渲染
*  在OpenGL中，GPU有2种渲染方式
  * On-Screen Rendering:当前屏幕渲染，在当前用于显示的屏幕缓冲区进行渲染操作
  * Off-Screen Rendering:离屏渲染，在当前屏幕缓冲区以外新开辟一个缓冲区进行渲染操作
*  离屏渲染消耗性能的原因
  * 需要创建新的缓冲区
  *  离屏渲染的整个过程，需要多次切换上下文环境，先是从当前屏幕(On-Screen)切换到离屏(Off-Screen);等到离屏渲染结束以后，将离屏缓冲区的渲染结果显示到屏幕上，又需要将上下文环境从离屏切换到当前屏幕 
*  哪些操作会触发离屏渲染?
  * 光栅化，layer.shouldRasterize = YES 
  * 遮罩，layer.mask
  * 圆角，同时设置layer.masksToBounds = YES、layer.cornerRadius大于0（考虑通过CoreGraphics绘制裁剪圆角，或者叫UI提供圆角图片）
  * 阴影，layer.shadowXXX   (如果设置了layer.shadowPath就不会产生离屏渲染)

### 内存泄露

#### 一、查找泄漏点 (两种工具)

​	

 + **1 > Analyze**

   ```
   - 学 名:  静态分析工具- 查 找:  可以通过 Product ->Analyze 菜单项启动- 快捷键:  CMD+shift +b.- Analyze主要分析以下四种问题:
     1) 逻辑错误：访问空指针或未初始化的变量等；
     2) 内存管理错误：如内存泄漏等；
     3) 声明错误：从未使用过的变量；
     4) Api调用错误：未包含使用的库和框架。
   ```

 + **2 >Instruments**

   ```
   - 学 名:   动态分析工具- 查 找:   Product ->Profile 菜单项启动- 快捷键:  CMD + i.
   简 介:它有很多跟踪模块可以动态分析和跟踪内存, CPU 和文件系统.
   ```



### 四、耗电优化

* 尽可能降低CPU、GPU功耗
*  少用定时器
*  优化I/O操作
  * 尽量不要频繁写入小数据，最好批量一次性写入
  *  读写大量重要数据时，考虑用dispatch_io，其提供了基于GCD的异步操作文件I/O的API。用dispatch_io系统会优化磁盘访问
  *  数据量比较大的，建议使用数据库(比如SQLite、CoreData)

* 网络优化

  * 减少、压缩网络数据
  *  如果多次请求的结果是相同的，尽量使用缓存
  *  使用断点续传，否则网络不稳定时可能多次传输相同的内容
  *  网络不可用时，不要尝试执行网络请求
  *  让用户可以取消长时间运行或者速度很慢的网络操作，设置合适的超时时间
  *  批量传输，比如，下载视频流时，不要传输很小的数据包，直接下载整个文件或者一大块一大块地下载。如果下载广告，一 次性多下载一些，然后再慢慢展示。如果下载电子邮件，一次下载多封，不要一封一封地下载

* 定位优化

  * 如果只是需要快速确定用户位置，最好用CLLocationManager的requestLocation方法。定位完成后，会自动让定位硬件断电
  *  如果不是导航应用，尽量不要实时更新位置，定位完毕就关掉定位服务
  *  尽量降低定位精度，比如尽量不要使用精度最高的kCLLocationAccuracyBest
  *  需要后台定位时，尽量设置pausesLocationUpdatesAutomatically为YES，如果用户不太可能移动的时候系统会自动暂停位置更新
  *  尽量不要使用startMonitoringSignificantLocationChanges，优先考虑startMonitoringForRegion:



  *  用户移动、摇晃、倾斜设备时，会产生动作(motion)事件，这些事件由加速度计、陀螺仪、磁力计等硬件检测。在不需要检测的场合，应该及时关闭这些硬件





### 五、参考资料

iOS启动优化博客：http://www.zoomfeng.com/blog/launch-time.html

dyld源码： https://opensource.apple.com/tarballs/dyld/		

dyld简介和分析：https://www.jianshu.com/p/be413358cd45

runtime源码：https://opensource.apple.com/source/objc4/

runtime源码分析：https://www.jianshu.com/p/3019605a4fc9

objc_msgSend源码： https://opensource.apple.com/source/objc4/objc4-723/runtime/Messengers.subproj/

开源项目fishhook: https://github.com/facebook/fishhook 

开源项目LSUR: https://github.com/tinymind/LSUnusedResources

AppCode官网：https://www.jetbrains.com/objc/

开源项目LinkMap: https://github.com/huanxsd/LinkMap

GNUstep源码 http://www.gnustep.org/resources/downloads.php

MachOView源码：<https://github.com/gdbinit/MachOView>

今日头条iOS客户端启动速度优化：<https://techblog.toutiao.com/2017/01/17/iosspeed/#more>

iOS 性能优化<https://github.com/skyming/iOS-Performance-Optimization>
