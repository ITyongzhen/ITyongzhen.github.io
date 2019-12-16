---
layout: post
title: iOS中load和initialize
date: 2017-08-16 15:32:24.000000000 +09:00
categories: 
- iOS
---

首发于[我的个人博客](https://ityongzhen.github.io/iOS%E4%B8%ADload%E5%92%8Cinitialize.html)

## +load方法

### 创建类和分类
- 先创建类YZPerson类，然后创建它的两个分类

YZPerson.m类

~~~~
#import "YZPerson.h"

@implementation YZPerson
+(void)run{
    NSLog(@"%s",__func__);
}
+(void)load{
    NSLog(@"%s",__func__);
}
@end
~~~~

YZPerson+test1.m类

~~~~
#import "YZPerson+test1.h"

@implementation YZPerson (test1)

+(void)run{
    NSLog(@"%s",__func__);
}
+(void)load{
    NSLog(@"%s",__func__);
}
@end
~~~~

YZPerson+test2.m类

~~~~
#import "YZPerson+test2.h"

@implementation YZPerson (test2)

+(void)run{
    NSLog(@"%s",__func__);
}
+(void)load{
    NSLog(@"%s",__func__);
}
@end
~~~~

创建完之后，这几个类不主动调用，直接启动

- 打印结果

~~~~
CateogryDemo[29670:414343] +[YZPerson load]
CateogryDemo[29670:414343] +[YZPerson(test1) load]
CateogryDemo[29670:414343] +[YZPerson(test2) load]
~~~~

这说明了。load方法，根本不需要我们自己调用，编译完成之后，就会调用。

### 疑问
但是有个疑问，因为，原来的类和分类中都写了load方法，为啥都调用呢？为什么不是只调用分类中的呢？

### 查看元类中的方法

#### 利用runtime写个打印方法

~~~~
void printMethodNamesOfClass(Class cls)
{
    unsigned int count;
    // 获得方法数组
    Method *methodList = class_copyMethodList(cls, &count);
    
    // 存储方法名
    NSMutableString *methodNames = [NSMutableString string];
    
    // 遍历所有的方法
    for (int i = 0; i < count; i++) {
        // 获得方法
        Method method = methodList[i];
        // 获得方法名
        NSString *methodName = NSStringFromSelector(method_getName(method));
        // 拼接方法名
        [methodNames appendString:methodName];
        [methodNames appendString:@", "];
    }
    
    // 释放
    free(methodList);
    
    // 打印方法名
    NSLog(@"%@ %@", cls, methodNames);
}
~~~~

#### 调用，因为是要查看类方法，所以需要打印元类中的方法

~~~~
- (void)viewDidLoad {
    [super viewDidLoad];
    printMethodNamesOfClass(object_getClass([YZPerson class]));
}
~~~~

#### 结果为

~~~~
CateogryDemo[30112:420944] +[YZPerson load]
CateogryDemo[30112:420944] +[YZPerson(test1) load]
CateogryDemo[30112:420944] +[YZPerson(test2) load]
CateogryDemo[30112:420944] YZPerson load, run, load, run, load, run,

~~~~

#### 小结

看得出来，有三个load方法，三个run方法

也进一步验证了，前面查看源码分析的结论：合并分类的时候，其方法列表等，不会覆盖掉原来类中的方法，是共存的。

### 源码分析

同上，先找到初始化方法

~~~~
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    
    // fixme defer initialization until an objc-using image is found?
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();

    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}

~~~~

查看 load_images 方法，这个是加载镜像，模块的方法

~~~~
// 加载镜像，模块的方法
void
load_images(const char *path __unused, const struct mach_header *mh)
{
    // Return without taking locks if there are no +load methods here.
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    // Discover load methods
    {
        mutex_locker_t lock2(runtimeLock);
        prepare_load_methods((const headerType *)mh);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
}

~~~~

查看加载load的代码

~~~~
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;

    loadMethodLock.assertLocked();

    // Re-entrant calls do nothing; the outermost call will finish the job.
    if (loading) return;
    loading = YES;

    void *pool = objc_autoreleasePoolPush();

    do {
        // 1. Repeatedly call class +loads until there aren't any more
        // 1. 调用类的load方法
        while (loadable_classes_used > 0) {
            call_class_loads();
        }

        // 2. Call category +loads ONCE
        // 2. 调用分类的load方法
        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories
    } while (loadable_classes_used > 0  ||  more_categories);

    objc_autoreleasePoolPop(pool);

    loading = NO;
}

~~~~

可以看出，是先调用类的load方法，再调用分类的load方法

继续跟代码

~~~~
static void call_class_loads(void)
{
    int i;
    
    // Detach current loadable list.
    struct loadable_class *classes = loadable_classes;
    int used = loadable_classes_used;
    loadable_classes = nil;
    loadable_classes_allocated = 0;
    loadable_classes_used = 0;
    
    // Call all +loads for the detached list.
    for (i = 0; i < used; i++) {
        Class cls = classes[i].cls;
        // 关键代码
        load_method_t load_method = (load_method_t)classes[i].method;
        if (!cls) continue; 

        if (PrintLoading) {
            _objc_inform("LOAD: +[%s load]\n", cls->nameForLogging());
        }
        (*load_method)(cls, SEL_load);
    }
    
    // Destroy the detached list.
    if (classes) free(classes);
}

~~~~

跟着上面注释中的关键代码，继续看源码

~~~~
typedef void(*load_method_t)(id, SEL);
~~~~

发现是一个指向函数地址的指针

### 小结
- 那么，答案就很清晰了。+load方法是根据方法地址直接调用，并不是经过objc_msgSend函数调用。



## 有分类和子类的情况
### 当有分类，也有子类，并且都有load方法 的情况下
例如YZStudent是 YZPerson 的子类
YZStudent+test1 是 YZStudent的分类
调用结果为

~~~~
CateogryDemo[31904:444099] +[YZPerson load]
CateogryDemo[31904:444099] +[YZStudent load]
CateogryDemo[31904:444099] +[YZStudent(test1) load]
CateogryDemo[31904:444099] +[YZPerson(test1) load]
CateogryDemo[31904:444099] +[YZPerson(test2) load]
~~~~


### 源码分析

接着之前的源码分析，继续查看源码

~~~~
// 加载镜像，模块的方法
void
load_images(const char *path __unused, const struct mach_header *mh)
{
    // Return without taking locks if there are no +load methods here.
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);

    // Discover load methods
    {
        mutex_locker_t lock2(runtimeLock);
        // prepare 准备工作
        prepare_load_methods((const headerType *)mh);
    }

    // Call +load methods (without runtimeLock - re-entrant)
    call_load_methods();
}
~~~~

~~~~
void prepare_load_methods(const headerType *mhdr)
{
    size_t count, i;

    runtimeLock.assertLocked();

    classref_t *classlist = 
        _getObjc2NonlazyClassList(mhdr, &count);
    for (i = 0; i < count; i++) {
        // 定制，规划
        schedule_class_load(remapClass(classlist[i]));
    }

    category_t **categorylist = _getObjc2NonlazyCategoryList(mhdr, &count);
    for (i = 0; i < count; i++) {
        category_t *cat = categorylist[i];
        Class cls = remapClass(cat->cls);
        if (!cls) continue;  // category for ignored weak-linked class
        realizeClass(cls);
        assert(cls->ISA()->isRealized());
        add_category_to_loadable_list(cat);
    }
}
~~~~

~~~~
static void schedule_class_load(Class cls)
{
    if (!cls) return;
    assert(cls->isRealized());  // _read_images should realize

    if (cls->data()->flags & RW_LOADED) return;

    // Ensure superclass-first ordering
    // 递归调用，传入父类
    schedule_class_load(cls->superclass);
    // 将cls 添加到 loadable_classes数组最后面
    add_class_to_loadable_list(cls);
    cls->setInfo(RW_LOADED); 
}
~~~~

可以看到，load方法，是递归调用，传入父类。依次加入数组中，那么调用的时候，先调用父类的load,再调用子类的load,而且和编译顺序无关。

### 总结

+load方法会在runtime加载类、分类时调用

每个类、分类的+load，在程序运行过程中只调用一次
load的调用顺序

1. 先调用类的load
	- 先编译的类，优先调用load
	- 调用子类的load之前，会先调用父类的load

2. 再调用分类的load
	- 先编译的分类，优先调用load


## +initialize方法
+initialize方法会在类第一次接收到消息时调用

### 调用顺序
先调用父类的+initialize，再调用子类的+initialize
(先初始化父类，再初始化子类，每个类只会初始化1次)

### 源码解读

[runtime源码](https://opensource.apple.com/tarballs/objc4/)
objc4源码解读过程

~~~~
>1. objc-msg-arm64.s
- objc_msgSend

>2. objc-runtime-new.mm
- class_getInstanceMethod
- lookUpImpOrNil
- lookUpImpOrForward
- _class_initialize
- callInitialize
- objc_msgSend(cls, SEL_initialize)
~~~~

具体来看代码

~~~~

Method class_getInstanceMethod(Class cls, SEL sel)
{
    if (!cls  ||  !sel) return nil;
    lookUpImpOrNil(cls, sel, nil, 
                   NO/*initialize*/, NO/*cache*/, YES/*resolver*/);


    return _class_getMethod(cls, sel);
}

~~~~

继续查看 lookUpImpOrNil

~~~~
IMP lookUpImpOrNil(Class cls, SEL sel, id inst, 
                   bool initialize, bool cache, bool resolver)
{
    IMP imp = lookUpImpOrForward(cls, sel, inst, initialize, cache, resolver);
    if (imp == _objc_msgForward_impcache) return nil;
    else return imp;
}
~~~~

lookUpImpOrForward 代码比较长，摘取关键代码如下

~~~~
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
 if (initialize  &&  !cls->isInitialized()) {
        runtimeLock.unlock();
        _class_initialize (_class_getNonMetaClass(cls, inst));
        runtimeLock.lock();

    }
 }
~~~~

继续查看关键代码

~~~~
void _class_initialize(Class cls)
{
    assert(!cls->isMetaClass());

    Class supercls;
    bool reallyInitialize = NO;

    if (supercls  &&  !supercls->isInitialized()) {
        _class_initialize(supercls);
    }
    
    // Try to atomically set CLS_INITIALIZING.
    {
        monitor_locker_t lock(classInitLock);
        if (!cls->isInitialized() && !cls->isInitializing()) {
            cls->setInitializing();
            reallyInitialize = YES;
        }
    }
    
    .....
    
    
     callInitialize(cls);
    
 }   
~~~~

这里可以看出，如果一个类，其父类没有初始化，就递归调用该方法进行初始化。最终调用callInitialize进行初始化

~~~~
void callInitialize(Class cls)
{
    ((void(*)(Class, SEL))objc_msgSend)(cls, SEL_initialize);
    asm("");
}


~~~~

最终，调用到objc_msgSend 方法，那我们继续看objc_msgSend源码，发现是汇编代码
截取部分如下:

~~~~
	.data
	.align 3
	.globl _objc_debug_taggedpointer_classes
_objc_debug_taggedpointer_classes:
	.fill 16, 8, 0
	.globl _objc_debug_taggedpointer_ext_classes
_objc_debug_taggedpointer_ext_classes:
	.fill 256, 8, 0
#endif

	ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame

	cmp	p0, #0			// nil check and tagged pointer check
#if SUPPORT_TAGGED_POINTERS
	b.le	LNilOrTagged		//  (MSB tagged pointer looks negative)
#else
	b.eq	LReturnZero
#endif
	ldr	p13, [x0]		// p13 = isa
	GetClassFromIsa_p16 p13		// p16 = class
LGetIsaDone:
	CacheLookup NORMAL		

~~~~

这也说明了，objc_msgSend性能高的原因，是因为直接操作汇编。

上述流程，用伪代码表示就是如下，其中
YZStudent 继承自 YZPerson

~~~~
if (YZStudent没有初始化) {
   if(YZPerson没有初始化){
      objc_msgSend([YZPerson class]，@selector(initialize));
    }
    objc_msgSend([YZStudent class]，@selector(initialize));
}
~~~~

### 注意点
+initialize和+load的很大区别是，+initialize是通过objc_msgSend进行调用的，所以有以下特点

- 如果子类没有实现+initialize，会调用父类的+initialize（所以父类的+initialize可能会被调用多次）
- 如果分类实现了+initialize，就覆盖类本身的+initialize调用

## 对比 +load 和 + initialize 的总结

### load、initialize方法的区别什么？
1. 调用方式
	- load是根据函数地址直接调用
	- initialize是通过objc_msgSend调用

2. 调用时刻
	- load是runtime加载类、分类的时候调用（只会调用1次）
	- initialize是类第一次接收到消息的时候调用，每一个类只会initialize一次（父类的initialize方法可能会被调用多次）

3. load、initialize的调用顺序？

	1. load
		1. 先调用类的load
			- 先编译的类，优先调用load
			- 调用子类的load之前，会先调用父类的load

		2. 再调用分类的load
			- 先编译的分类，优先调用load

	2. initialize
		1. 先初始化父类
		2. 再初始化子类（可能最终调用的是父类的initialize方法）		
		
本文相关代码github地址 [github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos)

本文参考资料:

[runtime源码](https://opensource.apple.com/tarballs/objc4/)

[iOS底层原理](https://ke.qq.com/course/package/11609)		
