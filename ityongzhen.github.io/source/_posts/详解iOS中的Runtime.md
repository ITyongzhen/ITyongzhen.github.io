---
layout: post
title: 详解iOS中的Runtime
date: 2017-10-15 15:32:24.000000000 +09:00
categories: 
- iOS
---


## 前言

### 导读

本文较长，分为以下几个部分

- isa
- class结构
- Type Encoding
- `cache_t`
- 方法调用
- 消息机制的三个阶段
- 消息发送
- 动态解析
- 消息转发
- 源码分析

什么是runtime

苹果官方说法

>The Objective-C language defers as many decisions as it can from compile time and link time to runtime.
（尽量将决定放到运行的时候，而不是在编译和链接过程）


### 版本和平台
runtime是有个两个版本的: legacy 、 modern
在Objective-C 1.0使用的是legacy，在2.0使用的是modern。这里简单介绍下区别：

- 在legacy runtime，如果你改变了实例变量的设计，需要重新编译它的子类。支持 32bit的OS X 程序
- 在modern runtime，如果你改变了实例变量的设计，不需要重新编译它的子类。支持iphone程序和OS X10.5之后的64bit程序



现在一般来说runtime都是指modern

## isa详解

### 共用体

要想学习Runtime，首先要了解它底层的一些常用数据结构，比如isa指针

在arm64架构之前，isa就是一个普通的指针，存储着Class、Meta-Class对象的内存地址

从arm64架构开始，对isa进行了优化，变成了一个共用体（union）结构，还使用位域来存储更多的信息。

查看[runtime源码](https://opensource.apple.com/tarballs/objc4/)可以看到关于isa结构。官方的源码是不能编译的。我自己编译了一份可以运行的源码在[github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos/tree/master/objc4-750-master)上。

~~~~
union isa_t {
    isa_t() { }
    isa_t(uintptr_t value) : bits(value) { }

    Class cls;
    uintptr_t bits;
    struct {
        ISA_BITFIELD;  // defined in isa.h
    };
};
~~~~

在runtime723版本以前，直接把结构体放在isa里面了。750版本之后，抽成宏了，展开宏`ISA_BITFIELD ` 在`__arm64__`架构下 如下所示

下面的代码对isa_t中的结构体进行了位域声明，地址从`nonpointer`起到`extra_rc`结束，从低到高进行排列。位域也是对结构体内存布局进行了一个声明，通过下面的结构体成员变量可以直接操作某个地址。位域总共占8字节，所有的位域加在一起正好是64位。

小提示：`union`中`bits`可以操作整个内存区，而位域只能操作对应的位。

eg: 一个对象的地址是`0x7faf1b580450` 转换成二进制`11111111010111100011011010110000000010001010000 `,然后根据不同位置，去匹配不同的含义

~~~~
 define ISA_BITFIELD                                                      \
      uintptr_t nonpointer        : 1;   //指针是否优化过                                   \
      uintptr_t has_assoc         : 1;   //是否有设置过关联对象，如果没有，释放时会更快                                   \
      uintptr_t has_cxx_dtor      : 1; 	 //是否有C++的析构函数（.cxx_destruct），如果没有，释放时会更快                                     \
      uintptr_t shiftcls          : 33; //存储着Class、Meta-Class对象的内存地址信息 \
      uintptr_t magic             : 6;  //用于在调试时分辨对象是否未完成初始化                                     \
      uintptr_t weakly_referenced : 1;  //是否有被弱引用指向过，如果没有，释放时会更快                                     \
      uintptr_t deallocating      : 1;  //对象是否正在释放                                     \
      uintptr_t has_sidetable_rc  : 1;  //引用计数器是否过大无法存储在isa中                                     \
      uintptr_t extra_rc          : 19 //里面存储的值是引用计数器减1
#   	define RC_ONE   (1ULL<<45)
#   	define RC_HALF  (1ULL<<18)
~~~~

### isa中不同的位域代表不同的含义。

- nonpointer
	- 0，代表普通的指针，存储着Class、Meta-Class对象的内存地址
	- 1，代表优化过，使用位域存储更多的信息

- has_assoc
	- 是否有设置过关联对象，如果没有，释放时会更快

- has_cxx_dtor
	- 是否有C++的析构函数（.cxx_destruct），如果没有，释放时会更快

- shiftcls
	- 存储着Class、Meta-Class对象的内存地址信息

- magic
	- 用于在调试时分辨对象是否未完成初始化

- weakly_referenced
	- 是否有被弱引用指向过，如果没有，释放时会更快
	
- deallocating
	- 对象是否正在释放

- extra_rc
	- 里面存储的值是引用计数器减1

- has_sidetable_rc
	- 引用计数器是否过大无法存储在isa中
	- 如果为1，那么引用计数会存储在一个叫SideTable的类的属性中


eg: 查看`objc_runtime-new.mm`文件中有如下代码。

~~~~
void *objc_destructInstance(id obj) 
{
    if (obj) {
        //是否有C++的析构函数
        bool cxx = obj->hasCxxDtor();
        //是否有设置过关联对象
        bool assoc = obj->hasAssociatedObjects();
        //有C++的析构函数，就去销毁
        if (cxx) object_cxxDestruct(obj);
         //有设置过关联对象，就去移除管理对象
        if (assoc) _object_remove_assocations(obj);
        
        obj->clearDeallocating();
    }

    return obj;
}
~~~~

可以看出，释放时候，会先判断是否有设置过关联对象，如果没有，释放时会更快。
是否有C++的析构函数（.cxx_destruct），如果没有，释放时会更快。其他的弱引用，nonpointer等，读者可自行看源码。

关Tagged Pointer技术，深入研究的话，可以参考唐巧博客[深入理解Tagged Pointer](https://blog.devtang.com/2014/05/30/understand-tagged-pointer/)

## class结构

用一幅图来表示

![](https://user-gold-cdn.xitu.io/2019/7/27/16c3202d8611c393?w=913&h=428&f=png&s=160446)

### objc_class

查看源码(只保留了主要代码)

~~~~
struct objc_object {
    Class _Nonnull isa  OBJC_ISA_AVAILABILITY;
};

struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;    //方法缓存
    class_data_bits_t bits;    // 用于获取具体的类的信息
}
~~~~

也就是说结构体`objc_class `里面

~~~~
struct objc_class : objc_object {
    Class isa;
    Class superclass;
    cache_t cache;    //方法缓存
    class_data_bits_t bits;    // 用于获取具体的类的信息}
~~~~

### `class_rw_t`

![](https://user-gold-cdn.xitu.io/2019/7/27/16c3202d8cb48997?w=870&h=251&f=png&s=52952)

根据bits可以得到`class_rw_t`，`class_rw_t`里面的methods、properties、protocols是二维数组，是可读可写的，包含了类的初始内容、分类的内容

eg:方法列表methods中存放着很多一维数组`method_list_t`,而每一个`method_list_t`中存放这`method_t`,`method_t`中是对应方法的imp指针，名字。类型等方法信息,在[详解iOS中分类Cateogry](https://ityongzhen.github.io/%E8%AF%A6%E8%A7%A3iOS%E4%B8%AD%E5%88%86%E7%B1%BBCateogry.html)一文中，我们知道，每个分类编译完成之后都会生成一个`_category_t`,对应着`method_list_t`。


~~~~

#define FAST_DATA_MASK          0x00007ffffffffff8UL

class_rw_t *data() { 
   return bits.data();
}

class_rw_t* data() {
   return (class_rw_t *)(bits & FAST_DATA_MASK);
}

~~~~

由代码可知 `bits & FAST_DATA_MASK`可获得`class_rw_t `.

~~~~
struct class_rw_t {
    // Be warned that Symbolication knows the layout of this structure.
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;

    method_array_t methods; //方法列表
    property_array_t properties; //属性列表
    protocol_array_t protocols; // 协议列表

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;
}
~~~~

结构体`method_array_t `

~~~~
class method_array_t : 
    public list_array_tt<method_t, method_list_t> 
{
    typedef list_array_tt<method_t, method_list_t> Super;

 public:
    method_list_t **beginCategoryMethodLists() {
        return beginLists();
    }
    
    method_list_t **endCategoryMethodLists(Class cls);

    method_array_t duplicate() {
        return Super::duplicate<method_array_t>();
    }
};

~~~~



### `method_t`

`method_t`是对方法、函数的封装

- IMP
	- IMP代表函数的具体实现
	
~~~~
// IMP代表函数的具体实现
typedef id _Nullable (*IMP)(id _Nonnull, SEL _Nonnull, ...); 
~~~~

- SEL

	- SEL代表方法、函数名，一般叫做选择器，底层结构跟`char *`类似
	- 可以通过`@selector()`和`sel_registerName()`获得
	- 可以通过`sel_getName()`和`NSStringFromSelector()`转成字符串
	- 不同类中相同名字的方法，所对应的方法选择器是相同的

~~~~
typedef struct objc_selector *SEL;
~~~~

- types包含了函数返回值、参数编码的字符串

	- 返回值 参数1	参数2	......	参数n
	- eg: `v16@0:8`代表，返回值void类型，第一个参数是id类型，第二个参数是SEL类型。后面会详细说明。
	
~~~~
struct method_t {
    SEL name; //函数名
    const char *types; //编码(返回值类型，参数类型)
    MethodListIMP imp; //指向函数的指针(函数地址)

    struct SortBySELAddress :
        public std::binary_function<const method_t&,
                                    const method_t&, bool>
    {
        bool operator() (const method_t& lhs,
                         const method_t& rhs)
        { return lhs.name < rhs.name; }
    };
};
~~~~

- 创建两个不同的类，并定义两个相同的方法，通过@selector()获取SEL并打印。可以发现SEL都是同一个对象，地址都是相同的。由此证明，不同类的相同SEL是同一个对象。

~~~~
@interface TestObject : NSObject
- (void)testMethod;
@end

@interface TestObject2 : NSObject
- (void)testMethod;
@end


// TestObject实现文件
@implementation TestObject
- (void)testMethod {
    NSLog(@"TestObject testMethod %p", @selector(testMethod));
}
@end


// TestObject2实现文件也一样
@implementation TestObject
- (void)testMethod {
    NSLog(@"TestObject testMethod %p", @selector(testMethod));
}

// 结果：
TestObject testMethod 0x100000f81
TestObject2 testMethod 0x100000f81
~~~~


### `class_ro_t`

![](https://user-gold-cdn.xitu.io/2019/7/27/16c3202d8cc20f9b?w=879&h=217&f=png&s=54481)

`class_ro_t`里面的baseMethodList、baseProtocols、ivars、baseProperties是一维数组，是只读的，包含了类的初始内容

~~~~
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;//instance对象占用的内存空间
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name; //类名
    method_list_t * baseMethodList; //方法列表
    protocol_list_t * baseProtocols; //协议列表
    const ivar_list_t * ivars; //成员变量列表

    const uint8_t * weakIvarLayout;
    property_list_t *baseProperties;

    method_list_t *baseMethods() const {
        return baseMethodList;
    }
};
~~~~

## Type Encoding

前面说了，`v16@0:8`代表，返回值void类型，第一个参数是id类型，第二个参数是SEL类型。这里详细说明

iOS中提供了一个叫做@encode的指令，可以将具体的类型表示成字符串编码，链接为 [Type Encodings
](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100-SW1)


![](https://user-gold-cdn.xitu.io/2019/7/27/16c3202d97dfc415?w=880&h=368&f=png&s=60370)


eg: 我们有如下函数

~~~~
void objc_msgSend(id receiver, SEL selector)
{
    if (receiver == nil) return;
    
    // 查找缓存
    ...
}
~~~~

就可以用`v16@0:8`表示

- 其中返回值void类型，第一个参数是id类型，第二个参数是SEL类型。
- 另外第一个数字16代表总共16个字节，0代表第一个参数从第0个字节开始，8代表第二个参数从第8个字节开始。
- 其实也可以简写为`v@:`，这在后面讲到消息转发的时候会用到。

再如：

~~~~
- (void)viewDidLoad {
    [super viewDidLoad];
    // Do any additional setup after loading the view.
    Method method = class_getClassMethod([self class], @selector(test));
    const char *str = method_getTypeEncoding(method);
    
    Method method2 = class_getClassMethod([self class], @selector(testWithNum:));
    const char *str2 = method_getTypeEncoding(method2);
    
    NSLog(@"test的类型 = %s ",str);
    NSLog(@"testWithNum: = %s ",str2);
}

// v16@0:8 
+(void)test{
    
}

// i20@0:8i16
+(int)testWithNum:(int)num{
    return num;
}
~~~~

输出结果为：

>RuntimeDemo[28247:303205] test的类型 = v16@0:8 
RuntimeDemo[28247:303205] testWithNum: = i20@0:8i16


对于方法`testWithNum`来说

- i表示返回值是int类型，20是参数总共20字节
- @表示第一个参数是id类型，0表示第一个参数从第0个字节开始 
- :表示第二个参数是SEL类型。8表示第二个参数从第8个字节开始。
- i表示第三个参数是int类型，16表示第三个参数从第16个字节开始
- 第三个参数从第16个字节开始，是Int类型，占用4字节。总共20字节

@encode




## 方法缓存`cache_t`

![](https://user-gold-cdn.xitu.io/2019/7/27/16c3202d97beb789?w=855&h=235&f=png&s=56387)

前面讲了Class内部结构,其中有个方法缓存`cache_t`，用散列表（哈希表）来缓存曾经调用过的方法，可以提高方法的查找速度

~~~~
struct cache_t {
    struct bucket_t *_buckets; //散列表
    mask_t _mask; //散列表的长度 -1
    mask_t _occupied; //已经缓存的方法数量
}
~~~~

散列表数组`_buckets `中存放着`bucket_t `,`bucket_t `的结构如下

~~~~

struct bucket_t {
     MethodCacheIMP _imp; //函数的内存地址
    cache_key_t _key;   //SEL作为Key
}
~~~~

### 散列表`cache_t`查找原理


在`cache_t`中如何查找方法，其实对于其他散列表也是通用的。

在文件`objc-cache.mm`中找到`bucket_t * cache_t::find(cache_key_t k, id receiver)`

~~~~
// 散列表中查找方法缓存
bucket_t * cache_t::find(cache_key_t k, id receiver)
{
    assert(k != 0);

    bucket_t *b = buckets();
    mask_t m = mask();
    mask_t begin = cache_hash(k, m);
    mask_t i = begin;
    do {
        if (b[i].key() == 0  ||  b[i].key() == k) {
            return &b[i];
        }
    } while ((i = cache_next(i, m)) != begin);

    // hack
    Class cls = (Class)((uintptr_t)this - offsetof(objc_class, cache));
    cache_t::bad_cache(receiver, (SEL)k, cls);
}


~~~~

其中，根据`key `和散列表长度减1 `mask` 计算出下标 `key & mask`，取出的值如果key和当初传进来的Key相同，就说明找到了。否则，就不是自己要找的方法，就有了hash冲突，把i的值加1，继续计算。如下代码

~~~~
// 计算下标
static inline mask_t cache_hash(cache_key_t key, mask_t mask) 
{
    return (mask_t)(key & mask);
}


//hash冲突的时候
static inline mask_t cache_next(mask_t i, mask_t mask) {
    return (i+1) & mask;
}
~~~~

### `cache_t `的扩容

当方法缓存太多的时候，超过了容量的3/4s时候，就需要扩容了。扩容是，把原来的容量增加为2倍


~~~~
static void cache_fill_nolock(Class cls, SEL sel, IMP imp, id receiver)
{
			...
    if (cache->isConstantEmptyCache()) {
        // Cache is read-only. Replace it.
        cache->reallocate(capacity, capacity ?: INIT_CACHE_SIZE);
    }
    else if (newOccupied <= capacity / 4 * 3) {
        // Cache is less than 3/4 full. Use it as-is.
    }
    else {
        // 来到这里说明，超过了3/4,需要扩容
        cache->expand();
    }

  		 ...
}

~~~~

具体扩容代码为

~~~~

enum {
    INIT_CACHE_SIZE_LOG2 = 2,
    INIT_CACHE_SIZE      = (1 << INIT_CACHE_SIZE_LOG2)
};

// cache_t的扩容
void cache_t::expand()
{
    cacheUpdateLock.assertLocked();
    
    uint32_t oldCapacity = capacity();
    // 扩容为原来的2倍
    uint32_t newCapacity = oldCapacity ? oldCapacity*2 : INIT_CACHE_SIZE;

    if ((uint32_t)(mask_t)newCapacity != newCapacity) {
        // mask overflow - can't grow further
        // fixme this wastes one bit of mask
        newCapacity = oldCapacity;
    }

    reallocate(oldCapacity, newCapacity);
}
~~~~


## 方法调用

我们经常写的OC方法调用，究竟是怎么个调用流程，怎么找到方法的呢？

我们如下代码

~~~~
Person *per = [[Person alloc]init];
[per test];
~~~~

执行指令

>clang -rewrite-objc main.m -o main.cpp

生成cpp文件，对应上面的代码为

~~~~
((void (*)(id, SEL))(void *)objc_msgSend)((id)per, sel_registerName("test"));
~~~~

简化为

~~~~
objc_msgSend)(per, sel_registerName("test"));
~~~~


其中，per称为消息接收者(receiver), test称为消息名称，也就是说，OC中方法的调用其实都是转换为objc_msgSend函数的调用


## 消息机制

### 三大阶段

OC中的方法调用，其实都是转换为`objc_msgSend`函数的调用

objc_msgSend的执行流程可以分为3大阶段

- 消息发送

- 动态方法解析

- 消息转发

运行时期，调用方法流程为

实例对象中存放 isa 指针以及实例变量，有 isa 指针可以找到实例对象所属的类对象 (类也是对象，面向对象中一切都是对象)，类中存放着实例方法列表，在这个方法列表中 SEL 作为 key，IMP 作为 value。 在编译时期，根据方法名字会生成一个唯一标识，这个标识就是 SEL。IMP 其实就是函数指针 指向了最终的函数实现。整个 Runtime 的核心就是 objc_msgSend 函数，通过给类发送 SEL 以传递消息，找到匹配的 IMP 再获取最终的实现

类中的 `super_class` 指针可以追溯整个继承链。向一个对象发送消息时，Runtime 会根据实例对象的 isa 指针找到其所属的类，并自底向上直至根类(NSObject)中 去寻找 SEL 所对应的方法，找到后就运行整个方法。

用一张经典的图来表示就是

![](https://user-gold-cdn.xitu.io/2019/7/27/16c3202d97f29bae?w=587&h=589&f=png&s=103456)


类中的 super_class 指针可以追溯整个继承链。向一个对象发送消息时，Runtime 会根据实例对象的 isa 指针找到其所属的类，并自底向上直至根类(NSObject)中 去寻找 SEL 所对应的方法，找到后就运行整个方法。

metaClass是元类，也有 isa 指针、super_class 指针。其中保存了类方法列表。

### 跟读源码顺序

`objc-msg-arm64.s`里面都是汇编
~~~~
objc-msg-arm64.s
ENTRY _objc_msgSend
b.le	LNilOrTagged
CacheLookup NORMAL
.macro CacheLookup
.macro CheckMiss
STATIC_ENTRY __objc_msgSend_uncached
.macro MethodTableLookup
__class_lookupMethodAndLoadCache3
~~~~

`objc-runtime-new.mm`

~~~~
objc-runtime-new.mm
_class_lookupMethodAndLoadCache3
lookUpImpOrForward
getMethodNoSuper_nolock、search_method_list、log_and_fill_cache
cache_getImp、log_and_fill_cache、getMethodNoSuper_nolock、log_and_fill_cache
_class_resolveInstanceMethod
_objc_msgForward_impcache
~~~~


一直跟到 `__forwarding__ `的时候，已经不开源的了。

~~~~
objc-msg-arm64.s
STATIC_ENTRY __objc_msgForward_impcache
ENTRY __objc_msgForward

Core Foundation
__forwarding__（不开源
~~~~

### `_objc_msgSend `

先来看 `objc-msg-arm64.s`

主要代码为

~~~~
    //1.进入objcmsgSend
	ENTRY _objc_msgSend
	UNWIND _objc_msgSend, NoFrame
    // x0 recevier
    // 消息接收者  消息名称
	cmp	p0, #0			// nil check and tagged pointer check
#if SUPPORT_TAGGED_POINTERS
    //2.isa 优化
	b.le	LNilOrTagged		//  (MSB tagged pointer looks negative)
#else
	b.eq	LReturnZero
#endif
	ldr	p13, [x0]		// p13 = isa
	GetClassFromIsa_p16 p13		// p16 = class
LGetIsaDone: // 3.isa优化完成
	CacheLookup NORMAL		//4.执行 CacheLookup NORMAL // calls imp or objc_msgSend_uncached

#if SUPPORT_TAGGED_POINTERS
LNilOrTagged:
	b.eq	LReturnZero		// nil check

...省略很多代码

.macro MethodTableLookup
	
	// push frame
	SignLR
	stp	fp, lr, [sp, #-16]!
	mov	fp, sp

	...省略很多代码
	
	// receiver and selector already in x0 and x1
	mov	x2, x16
	bl	__class_lookupMethodAndLoadCache3 //方法为_class_lookupMethodAndLoadCache3调用的汇编语言

	...省略很多代码
.endmacro

	STATIC_ENTRY __objc_msgSend_uncached
	UNWIND __objc_msgSend_uncached, FrameWithNoSaves

	// THIS IS NOT A CALLABLE C FUNCTION
	// Out-of-band p16 is the class to search
	
	MethodTableLookup //查找IMP
	TailCallFunctionPointer x17

	END_ENTRY __objc_msgSend_uncached


	STATIC_ENTRY __objc_msgLookup_uncached
	UNWIND __objc_msgLookup_uncached, FrameWithNoSaves

	// THIS IS NOT A CALLABLE C FUNCTION
	// Out-of-band p16 is the class to search
	
	MethodTableLookup
	ret

	END_ENTRY __objc_msgLookup_uncached


	STATIC_ENTRY _cache_getImp

	GetClassFromIsa_p16 p0
	CacheLookup GETIMP
	...省略很多代码
	
~~~~

从上面的代码可以看出方法查找 IMP 的工作交给了 OC 中的 _class_lookupMethodAndLoadCache3 函数，并将 IMP 返回（从 r11 挪到 rax）。最后在 objc_msgSend 中调用 IMP。

汇编代码比较晦涩难懂，因此这里将函数的实现反汇编成C语言的伪代码：

~~~~
/*
objc_msgSend的C语言版本伪代码实现.
receiver: 是调用方法的对象
op: 是要调用的方法名称字符串
*/
id  objc_msgSend(id receiver, SEL op, ...)
{

    //1............................ 对象空值判断。
    //如果传入的对象是nil则直接返回nil
    if (receiver == nil)
        return nil;
    
   //2............................ 获取或者构造对象的isa数据。
    void *isa = NULL;
    //如果对象的地址最高位为0则表明是普通的OC对象，否则就是Tagged Pointer类型的对象
    if ((receiver & 0x8000000000000000) == 0) {
        struct objc_object  *ocobj = (struct objc_object*) receiver;
        isa = ocobj->isa;
    }
    else { //Tagged Pointer类型的对象中没有直接保存isa数据，所以需要特殊处理来查找对应的isa数据。
        
        //如果对象地址的最高4位为0xF, 那么表示是一个用户自定义扩展的Tagged Pointer类型对象
        if (((NSUInteger) receiver) >= 0xf000000000000000) {
            
            //自定义扩展的Tagged Pointer类型对象中的52-59位保存的是一个全局扩展Tagged Pointer类数组的索引值。
            int  classidx = (receiver & 0xFF0000000000000) >> 52
            isa =  objc_debug_taggedpointer_ext_classes[classidx];
        }
        else {
            
            //系统自带的Tagged Pointer类型对象中的60-63位保存的是一个全局Tagged Pointer类数组的索引值。
            int classidx = ((NSUInteger) receiver) >> 60;
            isa  =  objc_debug_taggedpointer_classes[classidx];
        }
    }
    
   //因为内存地址对齐的原因和虚拟内存空间的约束原因，
   //以及isa定义的原因需要将isa与上0xffffffff8才能得到对象所属的Class对象。
    struct objc_class  *cls = (struct objc_class *)(isa & 0xffffffff8);
    
   //3............................ 遍历缓存哈希桶并查找缓存中的方法实现。
    IMP  imp = NULL;
    //cmd与cache中的mask进行与计算得到哈希桶中的索引，来查找方法是否已经放入缓存cache哈希桶中。
    int index =  cls->cache.mask & op;
    while (true) {
        
        //如果缓存哈希桶中命中了对应的方法实现，则保存到imp中并退出循环。
        if (cls->cache.buckets[index].key == op) {
              imp = cls->cache.buckets[index].imp;
              break;
        }
        
        //方法实现并没有被缓存，并且对应的桶的数据是空的就退出循环
        if (cls->cache.buckets[index].key == NULL) {
             break;
        }
        
        //如果哈希桶中对应的项已经被占用但是又不是要执行的方法，则通过开地址法来继续寻找缓存该方法的桶。
        if (index == 0) {
            index = cls->cache.mask;  //从尾部寻找
        }
        else {
            index--;   //索引减1继续寻找。
        }
    } /*end while*/

   //4............................ 执行方法实现或方法未命中缓存处理函数
    if (imp != NULL)
         return imp(receiver, op,  ...); //这里的... 是指传递给objc_msgSend的OC方法中的参数。
    else
         return objc_msgSend_uncached(receiver, op, cls, ...);
}

/*
  方法未命中缓存处理函数：objc_msgSend_uncached的C语言版本伪代码实现，这个函数也是用汇编语言编写。
*/
id objc_msgSend_uncached(id receiver, SEL op, struct objc_class *cls)
{
   //这个函数很简单就是直接调用了_class_lookupMethodAndLoadCache3 来查找方法并缓存到struct objc_class中的cache中，最后再返回IMP类型。
  IMP  imp =   _class_lookupMethodAndLoadCache3(receiver, op, cls);
  return imp(receiver, op, ....);
}

~~~~



## 消息发送阶段

前面跟到`_class_lookupMethodAndLoadCache3 `之后，后面就不是汇编了，是C语言的实现

- runtime的消息发送阶段，首先判断receiver是否为空，如果为空就直接返回
- 如果不为空，从receiverClass的缓存中，查找方法，如果找到了，就调用方法
- 如果没找到，就从receiverClass的`class_rw_t`中查找方法(分为二分查找和线性查找，两种),如果找到了，就结束查找，缓存一份到自己缓存中，调用方法
- 如果没找到，就去父类的缓存中查找，如果找到了，就就结束查找，缓存一份到自己缓存中，调用方法
- 如果没找到，就从父类的`class_rw_t`中查找方法,如果找到了，就结束查找，缓存一份到自己缓存中，调用方法
- 如果没找到，就看是否还有父类，如果有，就继续查父类的缓存，方法列表
- 如果没有父类，说明消息发送阶段结束，那么就进入第二阶段，动态方法解析阶段。

![](https://user-gold-cdn.xitu.io/2019/7/27/16c3202df83d016f?w=830&h=399&f=png&s=108439)


~~~~
IMP _class_lookupMethodAndLoadCache3(id obj, SEL sel, Class cls)
{
    
    return lookUpImpOrForward(cls, sel, obj, 
                              YES/*initialize*/, NO/*cache*/, YES/*resolver*/);
}
~~~~

关键代码在`lookUpImpOrForward `里面，下面的代码,增加了注释


~~~~
IMP lookUpImpOrForward(Class cls, SEL sel, id inst, 
                       bool initialize, bool cache, bool resolver)
{
    IMP imp = nil;
    bool triedResolver = NO; //是否动态解析过的标记
    runtimeLock.assertUnlocked();
    if (cache) {
        imp = cache_getImp(cls, sel);
        if (imp) return imp;
    }
    runtimeLock.lock();
    checkIsKnownClass(cls);

    if (!cls->isRealized()) {
        realizeClass(cls);
    }

    if (initialize  &&  !cls->isInitialized()) {
        runtimeLock.unlock();
        _class_initialize (_class_getNonMetaClass(cls, inst));
        runtimeLock.lock();
    }

    
 retry:    
    runtimeLock.assertLocked();

    // 这里先查缓存，虽然前面汇编里面已经查过了。但是有可能动态添加，导致缓存有更新
    imp = cache_getImp(cls, sel);
    //如果查到了，就直接跳转到最后
    if (imp) goto done;
    //来到这里，说明缓存没有
    // Try this class's method lists.
    { // 查找方法列表
        Method meth = getMethodNoSuper_nolock(cls, sel);
        if (meth) {
            log_and_fill_cache(cls, meth->imp, sel, inst, cls);
            imp = meth->imp;
            //如果查到了，就直接跳转到最后
            goto done;
        }
    }

    // Try superclass caches and method lists.
    { //查找父类的缓存和方法列表
        unsigned attempts = unreasonableClassCount();
        // for循环层层向上找
        for (Class curClass = cls->superclass;
             curClass != nil;
             curClass = curClass->superclass)
        {
            // Halt if there is a cycle in the superclass chain.
            if (--attempts == 0) {
                _objc_fatal("Memory corruption in class list.");
            }
            
            // Superclass cache.
            // 父类缓存
            imp = cache_getImp(curClass, sel);
            if (imp) {
                if (imp != (IMP)_objc_msgForward_impcache) {
                    // 如果父类缓存有，也要缓存一份到自己的缓存中
                    log_and_fill_cache(cls, imp, sel, inst, curClass);
                    //跳转到最后
                    goto done;
                }
                else {
                    break;
                }
            }
            
            // Superclass method list. 查找父类方法列表
            Method meth = getMethodNoSuper_nolock(curClass, sel);
            if (meth) {
                /// 如果父类方法列表有，也要缓存一份到自己的缓存中
                log_and_fill_cache(cls, meth->imp, sel, inst, curClass);
                imp = meth->imp;
                //如果查到了，就跳转到最后
                goto done;
            }
        }
    }

    // 来到这里，进入第二阶段，动态方法解析阶段,而且要求没有动态解析过
    if (resolver  &&  !triedResolver) {
        runtimeLock.unlock();
        _class_resolveMethod(cls, sel, inst);
        runtimeLock.lock();
        // Don't cache the result; we don't hold the lock so it may have 
        // changed already. Re-do the search from scratch instead.
        triedResolver = YES; //动态解析过，标记设为YES
        // 回到查找缓存的地方开始查找，缓存中没有加过，这次去查找，可以再方法列表中查到
        goto retry;
    }

    // No implementation found, and method resolver didn't help. 
    // Use forwarding.
    
    // 来到这里，说明进入第三阶段，消息转发阶段
    imp = (IMP)_objc_msgForward_impcache;
    cache_fill(cls, sel, imp, inst);

 done:
    runtimeLock.unlock();
    // 返回方法地址
    return imp;
}       
~~~~




## 动态方法解析

前面的消息发送阶段，没有找到，就来到动态方法解析阶段

头文件中定义两个方法

~~~~
- (void)test;
- (void)run;
~~~~

只实现test

~~~~
-(void)test{
    
    NSLog(@"%s",__func__);

}
~~~~

调用的是时候

~~~~
Person *per = [[Person alloc]init];
[per run];
~~~~

由前面的消息发送阶段知道，去查缓存，查方法列表，查父类等等，这些操作之后，都没有找到这个方法的实现，如果后面不做处理，必然抛出异常

报错方法找不到

> Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[Person run]: unrecognized selector sent to instance 0x100f436c0'

如果要处理的话，消息发送阶段处理不了。那么就来到第二阶段，动态解析阶段。这个阶段的处理，从前面的源码可知

~~~~
// 动态方法解析
void _class_resolveMethod(Class cls, SEL sel, id inst)
{
    if (! cls->isMetaClass()) { //如果不是元类对象
        // try [cls resolveInstanceMethod:sel]
        _class_resolveInstanceMethod(cls, sel, inst);
    } 
    else { // 是元类对象
        // try [nonMetaClass resolveClassMethod:sel]
        // and [cls resolveInstanceMethod:sel]
        _class_resolveClassMethod(cls, sel, inst);
        if (!lookUpImpOrNil(cls, sel, inst, 
                            NO/*initialize*/, YES/*cache*/, NO/*resolver*/)) 
        {
            _class_resolveInstanceMethod(cls, sel, inst);
        }
    }
}
~~~~

系统默认的`resolveClassMethod`和`resolveInstanceMethod `默认返回NO

~~~~
+ (BOOL)resolveClassMethod:(SEL)sel {
    return NO;
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    return NO;
}
~~~~

我们可以在动态解析阶段，重写`resolveInstanceMethod`并添加方法的实现

~~~~

+ (BOOL)resolveInstanceMethod:(SEL)sel
{
     if (sel == @selector(run)) {
             // 获取其他方法 实例方法，或者类方法都可以
            Method method = class_getInstanceMethod(self, @selector(test));

            // 动态添加test方法的实现
            class_addMethod(self, sel,
                            method_getImplementation(method),
                            method_getTypeEncoding(method));
			//等价于下面的
			// class_addMethod(self, sel,
                         method_getImplementation(method),
                         "v@:");
				
            // 返回YES代表有动态添加方法  其实这里返回NO，也是可以的，返回YES只是增加了一些打印
            return NO;
        }
        return [super resolveInstanceMethod:sel];
}

~~~~


上面的代码中，因为`-(void)test`无参无返回值，函数类型为`v@:`，所以，上面的`method_getTypeEncoding(method)`可以换成`"v@:"`也是没问题的。


这样的话，就相当于，调用run的时候，实际上调用的是test。由源码可知，动态解析完之后，回到查找缓存的地方开始查找，缓存中没有加过，这次去查找，可以再方法列表中查到。这样就可以正确执行了。输出结果为

>objc-test[6681:75992] -[Person test]

直接运行源码，如下图

![](https://user-gold-cdn.xitu.io/2019/7/27/16c3202e44698b78?w=1580&h=778&f=png&s=767693)

## 消息转发

如果前面消息发送和动态解析阶段，对方法都没有处理，我们还有最后一个阶段，消息转发阶段来处理。从源码的`imp = (IMP)_objc_msgForward_impcache;`可以看出，`_objc_msgForward_impcache `的代码是在汇编里面

~~~~ 

STATIC_ENTRY __objc_msgForward_impcache

// No stret specialization.
b	__objc_msgForward

END_ENTRY __objc_msgForward_impcache

	
ENTRY __objc_msgForward

adrp	x17, __objc_forward_handler@PAGE
// 这里进去之后，不开源了
ldr	p17, [x17, __objc_forward_handler@PAGEOFF]
TailCallFunctionPointer x17
... 还有很多代码

 
~~~~


跟到 `___forwarding___ `之后就不开源了

~~~~ 

objc-test[15568:163497] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[Person run]: unrecognized selector sent to instance 0x100f039f0'
*** First throw call stack:
(
	0   CoreFoundation                      0x00007fff307de063 __exceptionPreprocess + 250
	1   libobjc.A.dylib                     0x000000010038ec9f objc_exception_throw + 47
	2   CoreFoundation                      0x00007fff308671bd -[NSObject(NSObject) __retain_OA] + 0
	3   CoreFoundation                      0x00007fff307844b4 ___forwarding___ + 1427
	4   CoreFoundation                      0x00007fff30783e98 _CF_forwarding_prep_0 + 120
	5   objc-test                           0x0000000100000e11 main + 97
	6   libdyld.dylib                       0x00007fff672e93f9 start + 1
	7   ???                                 0x0000000000000001 0x0 + 1
)
libc++abi.dylib: terminating with uncaught exception of type NSException 
~~~~

在上述调用栈中，发现了在 Core Foundation 中会调用` ___forwarding___ `。根据资料也可以了解到，在 `objc_setForwardHandler` 时会传入 `__CF_forwarding_prep_0` 和 `___forwarding_prep_1___` 两个参数，而这两个指针都会调用` ____forwarding___ `。这个函数中，也交代了消息转发的逻辑

接下来怎么办呢？可以通过汇编调试，或逆向来进一步分析后续的实现。

>站在前人的代码上，能看的更远             ---鲁迅.尼古拉斯

### `___forwarding___ `的实现

国外有大神复原了`___forwarding___ `的实现，具体可参考[Hmmm, What's that Selector?](http://www.arigrant.com/blog/2013/12/13/a-selector-left-unhandled)

需要注意的是，复原了`___forwarding___ `的实现是伪代码。具体代码我已经放在了[github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos/tree/master/objc4-750-master)上。

~~~~ 
 伪代码
// 两个参数：前者为被转发消息的栈指针 IMP ，后者为是否返回结构体
int __forwarding__(void *frameStackPointer, int isStret) {
    id receiver = *(id *)frameStackPointer;
    SEL sel = *(SEL *)(frameStackPointer + 8);
    const char *selName = sel_getName(sel);
    Class receiverClass = object_getClass(receiver);
    
    // 调用 forwardingTargetForSelector:
    // 进入 备援接收 主要步骤
    if (class_respondsToSelector(receiverClass, @selector(forwardingTargetForSelector:))) {
        // 获得方法签名
        id forwardingTarget = [receiver forwardingTargetForSelector:sel];
        // 判断返回类型是否正确
        if (forwardingTarget && forwardingTarget != receiver) {
            if (isStret == 1) {
                int ret;
                objc_msgSend_stret(&ret,forwardingTarget, sel, ...);
                return ret;
            }
            return objc_msgSend(forwardingTarget, sel, ...);
        }
    }
    
    // 僵尸对象
    const char *className = class_getName(receiverClass);
    const char *zombiePrefix = "_NSZombie_";
    size_t prefixLen = strlen(zombiePrefix); // 0xa
    if (strncmp(className, zombiePrefix, prefixLen) == 0) {
        CFLog(kCFLogLevelError,
              @"*** -[%s %s]: message sent to deallocated instance %p",
              className + prefixLen,
              selName,
              receiver);
        <breakpoint-interrupt>
    }
    
    // 调用 methodSignatureForSelector 获取方法签名后再调用 forwardInvocation
    // 进入消息转发系统
    if (class_respondsToSelector(receiverClass, @selector(methodSignatureForSelector:))) {
        NSMethodSignature *methodSignature = [receiver methodSignatureForSelector:sel];
        // 判断返回类型是否正确
        if (methodSignature) {
            BOOL signatureIsStret = [methodSignature _frameDescriptor]->returnArgInfo.flags.isStruct;
            if (signatureIsStret != isStret) {
                CFLog(kCFLogLevelWarning ,
                      @"*** NSForwarding: warning: method signature and compiler disagree on struct-return-edness of '%s'.  Signature thinks it does%s return a struct, and compiler thinks it does%s.",
                      selName,
                      signatureIsStret ? "" : not,
                      isStret ? "" : not);
            }
            if (class_respondsToSelector(receiverClass, @selector(forwardInvocation:))) {
                // 传入消息的全部细节信息
                NSInvocation *invocation = [NSInvocation _invocationWithMethodSignature:methodSignature frame:frameStackPointer];
                
                [receiver forwardInvocation:invocation];
                
                void *returnValue = NULL;
                [invocation getReturnValue:&value];
                return returnValue;
            } else {
                CFLog(kCFLogLevelWarning ,
                      @"*** NSForwarding: warning: object %p of class '%s' does not implement forwardInvocation: -- dropping message",
                      receiver,
                      className);
                return 0;
            }
        }
    }
    
    SEL *registeredSel = sel_getUid(selName);
    
    // selector 是否已经在 Runtime 注册过
    if (sel != registeredSel) {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: selector (%p) for message '%s' does not match selector known to Objective C runtime (%p)-- abort",
              sel,
              selName,
              registeredSel);
    }  // doesNotRecognizeSelector，主动抛出异常
    // 表明未能得到处理
    else if (class_respondsToSelector(receiverClass,@selector(doesNotRecognizeSelector:))) {
        [receiver doesNotRecognizeSelector:sel];
    }
    else {
        CFLog(kCFLogLevelWarning ,
              @"*** NSForwarding: warning: object %p of class '%s' does not implement doesNotRecognizeSelector: -- abort",
              receiver,
              className);
    }
    
    // The point of no return.
    kill(getpid(), 9);
}
 
~~~~


### 小结

- 消息转发阶段，先判断`forwardingTargetForSelector `的返回值，如果有值，就向这个返回值发送消息。也就是`objc_msgSend(返回值, SEL)`。
- 如果返回为nil,就调用`methodSignatureForSelector`方法，如果有值，就调用`forwardInvocation`，其中的参数是一个 NSInvocation 对象，并将消息全部属性记录下来。 NSInvocation 对象包括了选择子、target 以及其他参数。其中的实现仅仅是改变了 target 指向，使消息保证能够调用。倘若发现本类无法处理，则继续想父类进行查找。直至 NSObject 。
- 如果`methodSignatureForSelector`方法返回nil,就调用`doesNotRecognizeSelector:`方法

![](https://user-gold-cdn.xitu.io/2019/7/27/16c3202e79fbcca6?w=825&h=383&f=png&s=59990)


上面都是源码分析，那下面代码验证


在源码中`forwardingTargetForSelector `系统默认返回nil 。

~~~~  
+ (id)forwardingTargetForSelector:(SEL)sel {
    return nil;
}

- (id)forwardingTargetForSelector:(SEL)sel {
    return nil;
}
~~~~

### 消息转发实例一

我们有类`Person`只定义了方法`- (void)run;`但是没有实现，另外有类`Car`，实现了方法`- (void)run;`



~~~~ 
@interface Car : NSObject
- (void)run;
@end


#import "Car.h"

@implementation Car
- (void)run{
    NSLog(@"%s",__func__);
}
@end
~~~~


在person中，重写`forwardingTargetForSelector `让返回`Car `对象

~~~~ 
// 消息转发
- (id)forwardingTargetForSelector:(SEL)aSelector{
    if (aSelector == @selector(run)) {
        return [[Car alloc] init];
    }
    return [super forwardingTargetForSelector:aSelector];
}
    
~~~~

调用的时候

~~~~

Person *per = [[Person alloc]init];
[per run]; 
 
~~~~


输出`objc-test[16694:174917] -[Car run]`


验证了前面说的，`forwardingTargetForSelector`返回值不为空的话，就向这个返回值发送消息，也就是 `objc_msgSend(返回值, SEL)`

### 消息转发实例二

如果前面的`forwardingTargetForSelector `返回为空， 就会调用 `methodSignatureForSelector` 获取方法签名后再调用 `forwardInvocation`


~~~~
// 方法签名：返回值类型、参数类型
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector
{
    if (aSelector == @selector(run)) {
       return [NSMethodSignature signatureWithObjCTypes:"v16@0:8"];
    }
     return [super methodSignatureForSelector:aSelector];
}
    
// NSInvocation封装了一个方法调用，包括：方法调用者、方法名、方法参数
//        anInvocation.target 方法调用者
//        anInvocation.selector 方法名
//        [anInvocation getArgument:NULL atIndex:0]
- (void)forwardInvocation:(NSInvocation *)anInvocation
{
        
   [anInvocation invokeWithTarget:[[Car alloc] init]];
}
~~~~   

依然可以调用到` -[Car run]`

**注意点1**

消息转发的`forwardingTargetForSelector `和`methodSignatureForSelector `以及`forwardInvocation `不仅支持实例方法，还支持类方法。不过系统没有提示，需要写成实例方法，然后把前面的`-`改成`+`即可。



**注意点2**
只能向运行时动态创建的类添加ivars，不能向已经存在的类添加ivars

这是因为在编译时只读结构体class_ro_t就会被确定，在运行时是不可更改的。ro结构体中有一个字段是instanceSize，表示当前类在创建对象时需要多少空间，后面的创建都根据这个size分配类的内存。
如果对一个已经存在的类增加一个参数，改变了ivars的结构，这样在访问改变之前创建的对象时，就会出现问题。


## 资料下载

### 资料下载
[github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos/tree/master/objc4-750-master)

### 参考资料


[runtime源码](https://opensource.apple.com/tarballs/objc4/)

[Hmmm, What's that Selector?](http://www.arigrant.com/blog/2013/12/13/a-selector-left-unhandled)

[RuntimePDF](https://github.com/DeveloperErenLiu/RuntimePDF)

[iOS底层原理](https://ke.qq.com/course/package/11609)

[Objective-C Runtime](https://hit-alibaba.github.io/interview/iOS/ObjC-Basic/Runtime.html)

[Objective-C 消息发送与转发机制原理](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)

[objc_msgSend消息传递学习笔记](https://www.desgard.com/iOS-Source-Probe/Objective-C/Runtime/objc_msgSend%E6%B6%88%E6%81%AF%E4%BC%A0%E9%80%92%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%20-%20%E5%AF%B9%E8%B1%A1%E6%96%B9%E6%B3%95%E6%B6%88%E6%81%AF%E4%BC%A0%E9%80%92%E6%B5%81%E7%A8%8B.html)


