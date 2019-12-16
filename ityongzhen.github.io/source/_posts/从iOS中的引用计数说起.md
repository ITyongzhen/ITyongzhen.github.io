---
layout: post
title: 从iOS中的引用计数说起
date: 2018-01-15 15:32:24.000000000 +09:00
categories: 
- iOS
---

本文首发于[个人博客](https://ityongzhen.github.io/iOS中的引用计数.html)


## 前言

维基百科中这么定义[引用计数](https://zh.wikipedia.org/wiki/%E5%BC%95%E7%94%A8%E8%AE%A1%E6%95%B0)
>引用计数是计算机编程语言中的一种内存管理技术，是指将资源（可以是对象、内存或磁盘空间等等）的被引用次数保存起来，当被引用次数变为零时就将其释放的过程。使用引用计数技术可以实现自动资源管理的目的。同时引用计数还可以指使用引用计数技术回收未使用资源的垃圾回收算法。

>当创建一个对象的实例并在堆上申请内存时，对象的引用计数就为1，在其他对象中需要持有这个对象时，就需要把该对象的引用计数加1，需要释放一个对象时，就将该对象的引用计数减1，直至对象的引用计数为0，对象的内存会被立刻释放。


## 在iOS中，使用引用计数来管理OC对象的内存

- 一个新创建的OC对象引用计数默认是1，当引用计数减为0，OC对象就会销毁，释放其占用的内存空间

- 调用retain会让OC对象的引用计数+1，调用release会让OC对象的引用计数-1

- 内存管理的经验总结
	- 当调用alloc、new、copy、mutableCopy方法返回了一个对象，在不需要这个对象时，要调用release或者autorelease来释放它
	- 想拥有某个对象，就让它的引用计数+1；不想再拥有某个对象，就让它的引用计数-1

- 可以通过以下私有函数来查看自动释放池的情况
	- `extern void _objc_autoreleasePoolPrint(void)`;


## isa

在[详解iOS中的Runtime](https://juejin.im/post/5d3be844f265da1bcd381fe3)一文中，对isa进行了详解。

这里进行简单概述

从arm64架构开始，苹果对isa进行了优化，变成了一个共用体（union）结构，还使用位域来存储更多的信息。如下

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


## Tagged Pointer

### 背景

再开始之前，先看这个代码

>NSNumber *num = @(20);

我们只有一个需要存储20这个数据，按照正常的技术方案，在64位CPU下，应该先去创建NSNumber对象，其值是20，然后再有个指向该地址的指针`num `。这样做存在什么问题呢？

- 内存浪费
	- 由于OC中的内存对齐，在64位下，创建一个对象至少16字节，再加上一个指针8个字节，总共24字节，也就是说，为了存储这个20而需要24字节，对内存方面是极大的浪费。

- 性能浪费
	- 为了存储和访问一个 NSNumber 对象，我们需要在堆上为其分配内存，另外还要维护它的引用计数，管理它的生命期。这些都给程序增加了额外的逻辑，造成运行效率上的损失


### Tagged Pointer技术

为了解决这个问题，苹果提出了Tagged Pointer的概念。对于 64 位程序，引入 Tagged Pointer 后，相关逻辑能减少一半的内存占用，以及 3 倍的访问速度提升，100 倍的创建、销毁速度提升。



- 从64bit开始，iOS引入了Tagged Pointer技术，用于优化NSNumber、NSDate、NSString等小对象的存储

- 在没有使用Tagged Pointer之前， NSNumber等对象需要动态分配内存、维护引用计数等，NSNumber指针存储的是堆中NSNumber对象的地址值

- 使用Tagged Pointer之后，NSNumber指针里面存储的数据变成了：Tag + Data，也就是将数据直接存储在了指针中

- 当指针不够存储数据时，才会使用动态分配内存的方式来存储数据

- `objc_msgSend`能识别Tagged Pointer，比如NSNumber的intValue方法，直接从指针提取数据，节省了以前的调用开销

- 如何判断一个指针是否为Tagged Pointer？
	- 最低有效位是1 (objc4-750之后)
	- 之前的版本(objc4-723以前)(iOS平台，最高有效位是1（第64bit）,Mac平台，最低有效位是1)
	
	
关于Tagged Pointer，想深入了解的，可以参照[深入理解 Tagged Pointer](https://www.infoq.cn/article/deep-understanding-of-tagged-pointer/)，就不在这赘述了。需要注意的是，之前的版本，变量的值直接存储在指针中，很容易的可以读取出来，例如`0xb000000000000012 ` 然而现在的版本中，苹果对这个指针做了一些编码处理，不能直接看出来是Tagged Pointer，例如`0x30a972fb5e339e15`然而它依然是Tagged Pointer，因为可以根据源码可知，是根据把它转为二进制之后最后一位是否为1来确定是否为Tagged Pointer。

~~~~

#if (TARGET_OS_OSX || TARGET_OS_IOSMAC) && __x86_64__
    // 64-bit Mac - tag bit is LSB
#   define OBJC_MSB_TAGGED_POINTERS 0
#else
    // Everything else - tag bit is MSB
#   define OBJC_MSB_TAGGED_POINTERS 1
#endif


#if OBJC_MSB_TAGGED_POINTERS
#   define _OBJC_TAG_MASK (1UL<<63)
#else
#   define _OBJC_TAG_MASK 1UL
#endif


static inline bool 
_objc_isTaggedPointer(const void * _Nullable ptr)
{
    return ((uintptr_t)ptr & _OBJC_TAG_MASK) == _OBJC_TAG_MASK;
}
~~~~
## 引用计数的存储

在64bit中，引用计数可以直接存储在优化过的isa指针中，也可能存储在SideTable类中,那`SideTable`中有什么呢？

`SideTable`的结构如下

~~~~
struct SideTable {
    spinlock_t slock;
    RefcountMap refcnts;//refcnts是一个存放着对象引用计数的散列表
    weak_table_t weak_table;

   	...还有很多代码
};
~~~~

其中 `RefcountMap refcnts`中存放着对象引用计数的散列表

## 获取引用计数

~~~~
// 引用计数
- (NSUInteger)retainCount {
    return ((id)self)->rootRetainCount();
}
~~~~

- `rootRetainCount`

~~~~
inline uintptr_t 
objc_object::rootRetainCount()
{
    //TaggedPointer不是一个普通的对象，不需要做引用计数的一些操作
    if (isTaggedPointer()) return (uintptr_t)this;

    sidetable_lock();
    isa_t bits = LoadExclusive(&isa.bits);
    ClearExclusive(&isa.bits);
    if (bits.nonpointer) { //优化过的isa
        uintptr_t rc = 1 + bits.extra_rc; // 这里进行了+1操作
        if (bits.has_sidetable_rc) {
            //能来到这里，说明引用计数不是存储在isa中，而是存储在sidetable中
            rc += sidetable_getExtraRC_nolock();
        }
        sidetable_unlock();
        return rc;
    }

    sidetable_unlock();
    return sidetable_retainCount();
}
~~~~

- `sidetable_getExtraRC_nolock `

~~~~
size_t 
objc_object::sidetable_getExtraRC_nolock()
{
    assert(isa.nonpointer);
    SideTable& table = SideTables()[this]; // this 就是key  根据这个key取出value
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()) return 0;
    else return it->second >> SIDE_TABLE_RC_SHIFT; // 取出的值 经过位运算之后返回
}
~~~~

`sidetable_retainCount() `方法的逻辑就是先从 `SideTable` 的静态方法获取当前实例对应的 `SideTable` 对象，其 `refcnts` 属性就是之前说的存储引用计数的散列表，然后在引用计数表中用迭代器查找当前实例对应的键值对，获取引用计数值，并在此基础上 +1 并将结果返回。这也就是为什么之前中说引用计数表存储的值为实际引用计数减一。

需要注意的是为什么这里把键值对的值做了向右移位操作`（it->second >> SIDE_TABLE_RC_SHIFT）`


## 引用计数的增删

在MRC 环境下可以使用 retain 和 release 方法对引用计数进行加一减一操作，它们分别调用了` _objc_rootRetain(id obj)` 和 `_objc_rootRelease(id obj)` 函数，不过后两者在 ARC 环境下也可使用。最后这两个函数又会调用 objc_object 的下面两个方法：

~~~~
inline id 
objc_object::rootRetain()
{
    assert(!UseGC);

    if (isTaggedPointer()) return (id)this;
    return sidetable_retain();
}

inline bool 
objc_object::rootRelease()
{
    assert(!UseGC);

    if (isTaggedPointer()) return false;
    return sidetable_release(true);
}
~~~~

就是先看释放支持isTaggedPointer，然后再操作 SideTable 中的 refcnts 属性，这与获取引用计数策略类似。sidetable_retain() 将 引用计数加一后返回对象，sidetable_release() 返回是否要执行 dealloc 方法：

### `引用计数的增加`

~~~~
ALWAYS_INLINE id 
objc_object::rootRetain(bool tryRetain, bool handleOverflow)
{
    if (isTaggedPointer()) return (id)this;

    bool sideTableLocked = false;
    bool transcribeToSideTable = false;

    isa_t oldisa;
    isa_t newisa;

    do {
        transcribeToSideTable = false;
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        if (slowpath(!newisa.nonpointer)) {
            ClearExclusive(&isa.bits);
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            if (tryRetain) return sidetable_tryRetain() ? (id)this : nil;
            else return sidetable_retain();
        }
        // don't check newisa.fast_rr; we already called any RR overrides
        if (slowpath(tryRetain && newisa.deallocating)) {
            ClearExclusive(&isa.bits);
            if (!tryRetain && sideTableLocked) sidetable_unlock();
            return nil;
        }
        uintptr_t carry;
        newisa.bits = addc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc++

        if (slowpath(carry)) {
            // newisa.extra_rc++ overflowed
            if (!handleOverflow) {
                ClearExclusive(&isa.bits);
                return rootRetain_overflow(tryRetain);
            }
            // Leave half of the retain counts inline and 
            // prepare to copy the other half to the side table.
            if (!tryRetain && !sideTableLocked) sidetable_lock();
            sideTableLocked = true;
            transcribeToSideTable = true;
            newisa.extra_rc = RC_HALF;
            newisa.has_sidetable_rc = true;
        }
    } while (slowpath(!StoreExclusive(&isa.bits, oldisa.bits, newisa.bits)));

    if (slowpath(transcribeToSideTable)) {
        // Copy the other half of the retain counts to the side table.
        sidetable_addExtraRC_nolock(RC_HALF);
    }

    if (slowpath(!tryRetain && sideTableLocked)) sidetable_unlock();
    return (id)this;
}
~~~~

- sidetable_retain

~~~~

id
objc_object::sidetable_retain()
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.nonpointer);
#endif
    SideTable& table = SideTables()[this];
    
    table.lock();
    size_t& refcntStorage = table.refcnts[this];
    if (! (refcntStorage & SIDE_TABLE_RC_PINNED)) {
        refcntStorage += SIDE_TABLE_RC_ONE;
    }
    table.unlock();

    return (id)this;
}

~~~~

### 引用计数的减少

~~~~
ALWAYS_INLINE bool 
objc_object::rootRelease(bool performDealloc, bool handleUnderflow)
{
    if (isTaggedPointer()) return false;

    bool sideTableLocked = false;

    isa_t oldisa;
    isa_t newisa;

 retry:
    do {
        oldisa = LoadExclusive(&isa.bits);
        newisa = oldisa;
        if (slowpath(!newisa.nonpointer)) {
            ClearExclusive(&isa.bits);
            if (sideTableLocked) sidetable_unlock();
            return sidetable_release(performDealloc);//引用计数减少
        }
        // don't check newisa.fast_rr; we already called any RR overrides
        uintptr_t carry;
        newisa.bits = subc(newisa.bits, RC_ONE, 0, &carry);  // extra_rc--
        if (slowpath(carry)) {
            // don't ClearExclusive()
            goto underflow;
        }
    } while (slowpath(!StoreReleaseExclusive(&isa.bits, 
                                             oldisa.bits, newisa.bits)));

    if (slowpath(sideTableLocked)) sidetable_unlock();
    return false;

 underflow:
    // newisa.extra_rc-- underflowed: borrow from side table or deallocate

    // abandon newisa to undo the decrement
    newisa = oldisa;
    ...还有很多代码
~~~~


- 函数`sidetable_release`

~~~~
uintptr_t
objc_object::sidetable_release(bool performDealloc)
{
#if SUPPORT_NONPOINTER_ISA
    assert(!isa.nonpointer);
#endif
    SideTable& table = SideTables()[this];

    bool do_dealloc = false;

    table.lock();
    RefcountMap::iterator it = table.refcnts.find(this);
    if (it == table.refcnts.end()) {
        do_dealloc = true;
        table.refcnts[this] = SIDE_TABLE_DEALLOCATING;
    } else if (it->second < SIDE_TABLE_DEALLOCATING) {
        // SIDE_TABLE_WEAKLY_REFERENCED may be set. Don't change it.
        do_dealloc = true;
        it->second |= SIDE_TABLE_DEALLOCATING;
    } else if (! (it->second & SIDE_TABLE_RC_PINNED)) {
        it->second -= SIDE_TABLE_RC_ONE;
    }
    table.unlock();
    if (do_dealloc  &&  performDealloc) {// 来到这里，说明引用计数为0，调用dealloc释放
        ((void(*)(objc_object *, SEL))objc_msgSend)(this, SEL_dealloc);
    }
    return do_dealloc;
}
~~~~

## 参考资料
[深入理解 Tagged Pointer](https://www.infoq.cn/article/deep-understanding-of-tagged-pointer/)

[详解iOS中的Runtime](https://juejin.im/post/5d3be844f265da1bcd381fe3)

[Runtime源码](https://opensource.apple.com/tarballs/objc4/)

[iOS底层原理](https://ke.qq.com/course/package/11609)


