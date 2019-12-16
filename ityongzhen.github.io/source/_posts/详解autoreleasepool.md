---
layout: post
title: 详解autoreleasepool
date: 2018-02-18 15:32:24.000000000 +09:00
categories: 
- iOS
---

本文首发于[个人博客](https://ityongzhen.github.io/%E8%AF%A6%E8%A7%A3autoreleasepool.html)

## 前言


文章开始之前，先想想下面三种场景，分别输出什么呢？

**注意`str`的长度不能太短**

**注意`str`的长度不能太短**

**注意`str`的长度不能太短**

~~~~
@interface ViewController ()
{
    __weak NSString *string_weak;
}

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景一
//    NSString *str =  [NSString stringWithFormat:@"https://ityongzhen.github.io/"];
//    string_weak = str;
    
    // 场景二
//    @autoreleasepool {
//        NSString *str =  [NSString stringWithFormat:@"https://ityongzhen.github.io/"];
//        string_weak = str;
//    }
//
//    // 场景三
    NSString *str = nil;
    @autoreleasepool {
        str =  [NSString stringWithFormat:@"https://ityongzhen.github.io/"];
        string_weak = str;
    }
    NSLog(@"string: %@ %s", string_weak,__func__);
}
- (void)viewWillAppear:(BOOL)animated{
    [super viewWillAppear:animated];
    NSLog(@"string: %@ %s", string_weak,__func__);
}

- (void)viewDidAppear:(BOOL)animated{
    [super viewDidAppear:animated];
    NSLog(@"string: %@ %s", string_weak,__func__);
}
~~~~


这个问题，暂时先放下，继续往下看。

## `autoreleasepool`生成c++文件

有如下代码

~~~~

@autoreleasepool {
     NSObject *obj = [[NSObject alloc] init];
}

~~~~

执行命令`xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc main.m -o main-arm64.cpp`生成c++文件，其对应的代码如下所示。

~~~~
{ __AtAutoreleasePool __autoreleasepool; 
        NSObject *obj = ((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)((NSObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSObject"), sel_registerName("alloc")), sel_registerName("init"));
    }
~~~~

简化一下也就是

~~~~
{
 __AtAutoreleasePool __autoreleasepool; 
  NSObject *obj = [[NSObject alloc] init];
 }
~~~~

其中`__AtAutoreleasePool`是什么呢？这是一个结构体，其内容如下，包含一个构造函数，在创建结构体的时候调用。一个析构函数，在结构体销毁的时候调用。

~~~~
struct __AtAutoreleasePool {
	//构造函数，在创建结构体的时候调用
  __AtAutoreleasePool() {
  		atautoreleasepoolobj = objc_autoreleasePoolPush();
  }
  
  //析构函数，在结构体销毁的时候调用
  ~__AtAutoreleasePool(){
 	 objc_autoreleasePoolPop(atautoreleasepoolobj);
  }
  
  void * atautoreleasepoolobj;
};
~~~~

所以，放在一起就是在开始的时候调用 `objc_autoreleasePoolPush()`结束时候调用` objc_autoreleasePoolPop(atautoreleasepoolobj)`

~~~~
struct __AtAutoreleasePool {
	//构造函数，在创建结构体的时候调用
  __AtAutoreleasePool() {
  		atautoreleasepoolobj = objc_autoreleasePoolPush();
  }
  	// 写在autoreleasepool内的代码
    NSObject *obj = [[NSObject alloc] init];
  
  //析构函数，在结构体销毁的时候调用
  ~__AtAutoreleasePool(){
 	 objc_autoreleasePoolPop(atautoreleasepoolobj);
  }
  
  void * atautoreleasepoolobj;
};
~~~~

## 源码分析


### `AutoreleasePoolPage`

具体源码可以再[Runtime源码](https://opensource.apple.com/tarballs/objc4/)中查看，从源码可以看到`objc_autoreleasePoolPush()`和`objc_autoreleasePoolPop `

~~~~
void *
objc_autoreleasePoolPush(void)
{
    return AutoreleasePoolPage::push();
}


void
objc_autoreleasePoolPop(void *ctxt)
{
    AutoreleasePoolPage::pop(ctxt);
}
~~~~

也就是说，这两个函数都是操作`AutoreleasePoolPage`来实现的。


类`AutoreleasePoolPage`中代码较多，筛选出主要代码如下

~~~~
class AutoreleasePoolPage 
{

    magic_t const magic;
    id *next;
    pthread_t const thread;
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t hiwat;
    
    id * begin() {
        return (id *) ((uint8_t *)this+sizeof(*this));
    }

    id * end() {
        return (id *) ((uint8_t *)this+SIZE);
    }

    bool empty() {
        return next == begin();
    }

    bool full() { 
        return next == end();
    }

    bool lessThanHalfFull() {
        return (next - begin() < (end() - begin()) / 2);
    }
    
    ...
}
~~~~

可以看出

AutoreleasePoolPage对象通过双向链表的形式连接在一起



其中

- magic 用来校验 AutoreleasePoolPage 的结构是否完整；
- next 指向最新添加的 autoreleased 对象的下一个位置，初始化时指向 begin() ；
- thread 指向当前线程；说明了，AutoreleasePoolPage和线程一一对应的。
- parent 指向父结点
- child 指向子结点
- depth 代表深度，从 0 开始，往后递增 1；
- hiwat 代表 high water mark 。



### 每个AutoreleasePoolPage对象占用4096字节


- 每个AutoreleasePoolPage对象占用4096字节内存，除了用来存放它内部的成员变量，剩下的空间用来存放autorelease对象的地址



~~~~
#define I386_PGBYTES            4096            /* bytes per 80386 page */

#define PAGE_SIZE               I386_PGBYTES

static size_t const SIZE = PAGE_MAX_SIZE


  id * begin() {
        return (id *) ((uint8_t *)this+sizeof(*this));
    }

    id * end() {
        return (id *) ((uint8_t *)this+SIZE);
 }
~~~~

从上面的源码中可以看出来

- 每个`AutoreleasePoolPage `有是4096字节,
- 以及`begin`指向的是开始存放`autorelease对象`的地方，
- `end指向结尾的位置`

## AutoreleasePoolPage存不下了怎么办？

如果一个`AutoreleasePoolPage `存不下了，就会再创建一个`AutoreleasePoolPage对象`，第一个`AutoreleasePoolPage对象`的`child`指向第二个`AutoreleasePoolPage对象`，第二个`AutoreleasePoolPage对象`的`parent`指向第一个`AutoreleasePoolPage对象`。图形表示就是如下

![](https://user-gold-cdn.xitu.io/2019/9/4/16cfb77743f3dd0e?w=1548&h=592&f=png&s=95929)

## `push`、`pop`、`autorelease`

`AutoreleasePoolPage`里面有`push`和`pop`函数

- 调用`push`方法会将一个`POOL_BOUNDARY`入栈，并且返回其存放的内存地址

- 调用`pop`方法时传入一个`POOL_BOUNDARY`的内存地址，会从最后一个入栈的对象开始发送release消息，直到遇到这个`POOL_BOUNDARY`

- `id *next`指向了下一个能存放`autorelease`对象地址的区域  

### `push`

~~~~
  static inline void *push() 
    {
        id *dest;
        if (DebugPoolAllocation) {
            // Each autorelease pool starts on a new pool page.
            dest = autoreleaseNewPage(POOL_BOUNDARY);
        } else {
            dest = autoreleaseFast(POOL_BOUNDARY);
        }
        assert(dest == EMPTY_POOL_PLACEHOLDER || *dest == POOL_BOUNDARY);
        return dest;
    }
    

~~~~

~~~~
 static inline id *autoreleaseFast(id obj)
    {
        AutoreleasePoolPage *page = hotPage();
        if (page && !page->full()) {//page没有满，就把obj对象加到page
            return page->add(obj);
        } else if (page) {//page满了 创建新的page
            return autoreleaseFullPage(obj, page);
        } else {
            return autoreleaseNoPage(obj);
        }
    }
~~~~

### `pop`

~~~~
  static inline void pop(void *token) 
    {
        AutoreleasePoolPage *page;
        id *stop;

        if (token == (void*)EMPTY_POOL_PLACEHOLDER) {
            // Popping the top-level placeholder pool.
            if (hotPage()) {
                // Pool was used. Pop its contents normally.
                // Pool pages remain allocated for re-use as usual.
                pop(coldPage()->begin());
            } else {
                // Pool was never used. Clear the placeholder.
                setHotPage(nil);
            }
            return;
        }

        page = pageForPointer(token);
        stop = (id *)token;
        if (*stop != POOL_BOUNDARY) {
            if (stop == page->begin()  &&  !page->parent) {
                // Start of coldest page may correctly not be POOL_BOUNDARY:
                // 1. top-level pool is popped, leaving the cold page in place
                // 2. an object is autoreleased with no pool
            } else {
                // Error. For bincompat purposes this is not 
                // fatal in executables built with old SDKs.
                return badPop(token);
            }
        }

        if (PrintPoolHiwat) printHiwat();

        page->releaseUntil(stop);

        // memory: delete empty children
        if (DebugPoolAllocation  &&  page->empty()) {
            // special case: delete everything during page-per-pool debugging
            AutoreleasePoolPage *parent = page->parent;
            page->kill();
            setHotPage(parent);
        } else if (DebugMissingPools  &&  page->empty()  &&  !page->parent) {
            // special case: delete everything for pop(top) 
            // when debugging missing autorelease pools
            page->kill();
            setHotPage(nil);
        } 
        else if (page->child) {
            // hysteresis: keep one empty child if page is more than half full
            if (page->lessThanHalfFull()) {
                page->child->kill();
            }
            else if (page->child->child) {
                page->child->child->kill();
            }
        }
    }

~~~~

### `autorelease`

~~~~
inline id 
objc_object::autorelease()
{
    if (isTaggedPointer()) return (id)this;
    // 调用rootAutorelease
    if (fastpath(!ISA()->hasCustomRR())) return rootAutorelease();

    return ((id(*)(objc_object *, SEL))objc_msgSend)(this, SEL_autorelease);
}
~~~~

`rootAutorelease `调用`rootAutorelease2 `

~~~~
inline id 
objc_object::rootAutorelease()
{
    if (isTaggedPointer()) return (id)this;
    if (prepareOptimizedReturn(ReturnAtPlus1)) return (id)this;

    return rootAutorelease2();
}
~~~~

`rootAutorelease2 `调用`autorelease `

~~~~
__attribute__((noinline,used))
id 
objc_object::rootAutorelease2()
{
    assert(!isTaggedPointer());
    return AutoreleasePoolPage::autorelease((id)this);
}
~~~~

`autorelease `

~~~~
static inline id autorelease(id obj)
{
    assert(obj);
    assert(!obj->isTaggedPointer());
    //再次调用autoreleaseFast
    id *dest __unused = autoreleaseFast(obj);
    assert(!dest  ||  dest == EMPTY_POOL_PLACEHOLDER  ||  *dest == obj);
    return obj;
}
~~~~

## `POOL_BOUNDARY`

上面的源码可以发现`POOL_BOUNDARY`是个很重要的角色，相当于一个哨兵，

- 每当进行一次`objc_autoreleasePoolPush`调用时，runtime向当前的`AutoreleasePoolPage`中add进一个哨兵对象(POOL_BOUNDARY)，值为0（也就是个nil）
- `objc_autoreleasePoolPush`的返回值正是这个哨兵对象的地址，被`objc_autoreleasePoolPop`(哨兵对象)作为入参，于是
	- 根据传入的哨兵对象地址找到哨兵对象所处的page
	- 在当前page中，将晚于哨兵对象插入的所有autorelease对象都发送一次`- release`消息，并向回移动next指针到正确位置
	- 从最新加入的对象一直向前清理，可以向前跨越若干个page，直到哨兵所在的page

## `@autoreleasepool`的嵌套

如果多个`@autoreleasepool`嵌套会怎么样呢？

### 打印

源码中有如下代码

~~~~
void 
_objc_autoreleasePoolPrint(void)
{
    AutoreleasePoolPage::printAll();
}


 static void printAll()
    {        
        _objc_inform("##############");
        _objc_inform("AUTORELEASE POOLS for thread %p", pthread_self());

        AutoreleasePoolPage *page;
        ptrdiff_t objects = 0;
        for (page = coldPage(); page; page = page->child) {
            objects += page->next - page->begin();
        }
        _objc_inform("%llu releases pending.", (unsigned long long)objects);

        if (haveEmptyPoolPlaceholder()) {
            _objc_inform("[%p]  ................  PAGE (placeholder)", 
                         EMPTY_POOL_PLACEHOLDER);
            _objc_inform("[%p]  ################  POOL (placeholder)", 
                         EMPTY_POOL_PLACEHOLDER);
        }
        else {
            for (page = coldPage(); page; page = page->child) {
                page->print();
            }
        }

        _objc_inform("##############");
    }
~~~~

也就是说`_objc_autoreleasePoolPrint `函数可以用来打印一些日志

### 一层`@autoreleasepool`

~~~~
extern void _objc_autoreleasePoolPrint(void);

int main(int argc, char * argv[]) {
    @autoreleasepool {

        _objc_autoreleasePoolPrint();
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
~~~~



输出如下，只有一个哨兵对象(POOL)

~~~~
objc[32644]: ##############
objc[32644]: AUTORELEASE POOLS for thread 0x11b111d40
objc[32644]: 3 releases pending.
objc[32644]: [0x7fcabf802000]  ................  PAGE  (hot) (cold)
objc[32644]: [0x7fcabf802038]    0x600003f70500  __NSArrayI
objc[32644]: [0x7fcabf802040]    0x600000950f00  __NSSetI
objc[32644]: [0x7fcabf802048]  ################  POOL 0x7fcabf802048
objc[32644]: ##############

~~~~


### 三层`@autoreleasepool`

如果有三个` @autoreleasepool`呢？

~~~~
extern void _objc_autoreleasePoolPrint(void);

int main(int argc, char * argv[]) {
    @autoreleasepool {
        @autoreleasepool {
            @autoreleasepool {
                 _objc_autoreleasePoolPrint();
            }
        }
       
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
~~~~

输出如下，有三个`POOL `，说明有三个哨兵。

~~~~
objc[32735]: ##############
objc[32735]: AUTORELEASE POOLS for thread 0x114f02d40
objc[32735]: 5 releases pending.
objc[32735]: [0x7f91fd005000]  ................  PAGE  (hot) (cold)
objc[32735]: [0x7f91fd005038]    0x600001bbd380  __NSArrayI
objc[32735]: [0x7f91fd005040]    0x600002da4eb0  __NSSetI
objc[32735]: [0x7f91fd005048]  ################  POOL 0x7f91fd005048
objc[32735]: [0x7f91fd005050]  ################  POOL 0x7f91fd005050
objc[32735]: [0x7f91fd005058]  ################  POOL 0x7f91fd005058
objc[32735]: ##############

~~~~

### 销毁一个` @autoreleasepool`

如果上面代码中最里面的` @autoreleasepool`退出之后再打印呢？

~~~~
extern void _objc_autoreleasePoolPrint(void);

int main(int argc, char * argv[]) {
    @autoreleasepool {
        @autoreleasepool {
            @autoreleasepool {
                
            }
            //打印的时候，最里面的@autoreleasepool已经退出了
             _objc_autoreleasePoolPrint();
        }
       
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
~~~~

输出为如下，只有两个哨兵(POOL)

~~~~
objc[32812]: ##############
objc[32812]: AUTORELEASE POOLS for thread 0x1142b1d40
objc[32812]: 4 releases pending.
objc[32812]: [0x7fe86e800000]  ................  PAGE  (hot) (cold)
objc[32812]: [0x7fe86e800038]    0x600003edb0c0  __NSArrayI
objc[32812]: [0x7fe86e800040]    0x6000008dd680  __NSSetI
objc[32812]: [0x7fe86e800048]  ################  POOL 0x7fe86e800048
objc[32812]: [0x7fe86e800050]  ################  POOL 0x7fe86e800050
objc[32812]: ##############
~~~~

进一步说明了，

- 调用push方法会将一个`POOL_BOUNDARY`入栈，并且返回其存放的内存地址

- 调用pop方法时传入一个`POOL_BOUNDARY`的内存地址，会从最后一个入栈的对象开始发送`release`消息，直到遇到这个`POOL_BOUNDARY`


## 开头的问题

现在我们回过头看文章开头的问题，应该很好回答了。



~~~~
@interface ViewController ()
{
    __weak NSString *string_weak;
}

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景一
//    NSString *str =  [NSString stringWithFormat:@"https://ityongzhen.github.io/"];
//    string_weak = str;
    
    // 场景二
//    @autoreleasepool {
//        NSString *str =  [NSString stringWithFormat:@"https://ityongzhen.github.io/"];
//        string_weak = str;
//    }
//
//    // 场景三
    NSString *str = nil;
    @autoreleasepool {
        str =  [NSString stringWithFormat:@"https://ityongzhen.github.io/"];
        string_weak = str;
    }
    NSLog(@"string: %@ %s", string_weak,__func__);
}
- (void)viewWillAppear:(BOOL)animated{
    [super viewWillAppear:animated];
    NSLog(@"string: %@ %s", string_weak,__func__);
}

- (void)viewDidAppear:(BOOL)animated{
    [super viewDidAppear:animated];
    NSLog(@"string: %@ %s", string_weak,__func__);
}
~~~~


### 输出结果如下：

~~~~
// 场景一
iOS定时器[24714:332118] string: https://ityongzhen.github.io/ -[ViewController viewDidLoad]
iOS定时器[24714:332118] string: https://ityongzhen.github.io/ -[ViewController viewWillAppear:]
iOS定时器[24714:332118] string: (null) -[ViewController viewDidAppear:]

场景二
iOS定时器[24676:331168] string: (null) -[ViewController viewDidLoad]
iOS定时器[24676:331168] string: (null) -[ViewController viewWillAppear:]
iOS定时器[24676:331168] string: (null) -[ViewController viewDidAppear:]

场景三
iOS定时器[24505:328544] string: https://ityongzhen.github.io/ -[ViewController viewDidLoad]
iOS定时器[24505:328544] string: (null) -[ViewController viewWillAppear:]
iOS定时器[24505:328544] string: (null) -[ViewController viewDidAppear:]
~~~~

### 场景一

  当使用 `[NSString stringWithFormat:@"https://ityongzhen.github.io/"]` 创建一个对象时，这个对象的引用计数为 1 ，并且这个对象被系统自动添加到了当前的 autoreleasepool 中。当使用局部变量 str 指向这个对象时，这个对象的引用计数 +1 ，变成了 2 。因为在 ARC 下 `NSString *str `本质上就是 `__strong NSString *str` 。所以在 `viewDidLoad` 方法返回前，这个对象是一直存在的，且引用计数为 2 。而当` viewDidLoad` 方法返回时，局部变量 `str` 被回收，指向了 nil 。因此，其所指向对象的引用计数 -1 ，变成了 1 。

而在 `viewWillAppear` 方法中，我们仍然可以打印出这个对象的值，在`viewDidAppear `方法中，这个值为空，这个就要牵扯到RunLoop的知识了。[详解RunLoop之源码分析](https://juejin.im/post/5d04c88d5188255e1305ca09)一文讲述了RunLoop的底层，这里说一下，我们的iOS处理事件是以RunLoop一直循环执行的。`viewDidLoad `和`viewWillAppear `在同一个RunLoop循环中，所以在 `viewWillAppear` 方法中，我们仍然可以打印出这个对象的值，但是`viewDidLoad `的时候，那个RunLoop循环已经执行完了，这个对象才被彻底的释放。

### 场景二

当通过 `[NSString stringWithFormat:@"https://ityongzhen.github.io/"]` 创建一个对象时，这个对象的引用计数为 1 。而当使用局部变量 str 指向这个对象时，这个对象的引用计数 +1 ，变成了 2 。而出了当前作用域时，局部变量 str 变成了 nil ，所以其所指向对象的引用计数变成 1 。另外，我们知道当出了 `@autoreleasepool {} `的作用域时，当前 `autoreleasepool` 被 drain ，其中的 autoreleased 对象被 release 。所以这个对象的引用计数变成了 0 ，对象最终被释放

### 场景三


当出了 `@autoreleasepool {}` 的作用域时，其中的 `autoreleased` 对象被 release ，对象的引用计数变成 1 。当出了局部变量 str 的作用域，即 `viewDidLoad` 方法返回时，str 指向了 nil ，其所指向对象的引用计数变成 0 ，对象最终被释放

### 注意点

前面说了**注意`str`的长度不能太短**是为什么呢？


是因为如果`str`过短。例如

~~~~
@interface ViewController ()
{
    __weak NSString *string_weak;
}

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    // 场景一
//    NSString *str =  [NSString stringWithFormat:@"abc"];
//    string_weak = str;
    
    // 场景二
//    @autoreleasepool {
//        NSString *str =  [NSString stringWithFormat:@"abc"];
//        string_weak = str;
//    }
//
//    // 场景三
    NSString *str = nil;
    @autoreleasepool {
        str =  [NSString stringWithFormat:@"abc"];
        string_weak = str;
    }
    NSLog(@"string: %@ %s", string_weak,__func__);
}
- (void)viewWillAppear:(BOOL)animated{
    [super viewWillAppear:animated];
    NSLog(@"string: %@ %s", string_weak,__func__);
}

- (void)viewDidAppear:(BOOL)animated{
    [super viewDidAppear:animated];
    NSLog(@"string: %@ %s", string_weak,__func__);
}
~~~~

结果如下：

~~~~
// 场景一
iOS定时器[24714:332118] string: abc -[ViewController viewDidLoad]
iOS定时器[24714:332118] string: abc -[ViewController viewWillAppear:]
iOS定时器[24714:332118] string: abc -[ViewController viewDidAppear:]

场景二
iOS定时器[24676:331168] string: abc -[ViewController viewDidLoad]
iOS定时器[24676:331168] string: abc -[ViewController viewWillAppear:]
iOS定时器[24676:331168] string: abc -[ViewController viewDidAppear:]

场景三
iOS定时器[24505:328544] string: abc -[ViewController viewDidLoad]
iOS定时器[24505:328544] string: abc -[ViewController viewWillAppear:]
iOS定时器[24505:328544] string: abc -[ViewController viewDidAppear:]
~~~~

这是因为，字符串的`abc`采用的是Tagged Pointer技术，不是一个标准的OC对象。不存在说再堆上开辟空间存储对象什么的。关于Tagged Pointer可以参考这篇文章[iOS中的引用计数](https://juejin.im/post/5d5a47f051882521610014a4),这里不做赘述。

## 总结

- 自动释放池的主要底层数据结构是：`__AtAutoreleasePool`、`AutoreleasePoolPage`

- 调用了`autorelease`的对象最终都是通过`AutoreleasePoolPage`对象来管理的

- 每个`AutoreleasePoolPage`对象占用4096字节内存，除了用来存放它内部的成员变量，剩下的空间用来存放`autorelease`对象的地址
- 所有的`AutoreleasePoolPage`对象通过双向链表的形式连接在一起

- 调用`push`方法会将一个`POOL_BOUNDARY`入栈，并且返回其存放的内存地址

- 调用`pop`方法时传入一个`POOL_BOUNDARY`的内存地址，会从最后一个入栈的对象开始发送`release`消息，直到遇到这个`POOL_BOUNDARY`

- `id *next`指向了下一个能存放`autorelease`对象地址的区域 

- iOS在主线程的`Runloop`中注册了2个`Observer`
	- 第1个`Observer`监听了`kCFRunLoopEntry`事件，会调用`objc_autoreleasePoolPush()`
	- 第2个`Observer`
		- 监听了`kCFRunLoopBeforeWaiting`事件，会调用`objc_autoreleasePoolPop()`、`objc_autoreleasePoolPush()`
		- 监听了`kCFRunLoopBeforeExit`事件，会调用`objc_autoreleasePoolPop()`
- 在当次RunLoop将要结束的时候，调用`objc_autoreleasePoolPop()`

## 参考资料



[黑幕背后的Autorelease](http://blog.sunnyxx.com/2014/10/15/behind-autorelease/)

[Runtime源码](https://opensource.apple.com/tarballs/objc4/)

[Objective-C Autorelease Pool 的实现原理](https://blog.leichunfeng.com/blog/2015/05/31/objective-c-autorelease-pool-implementation-principle/)


[详解RunLoop之源码分析](https://juejin.im/post/5d04c88d5188255e1305ca09)

[iOS底层原理](https://ke.qq.com/course/package/11609)

[iOS中的引用计数](https://juejin.im/post/5d5a47f051882521610014a4)

