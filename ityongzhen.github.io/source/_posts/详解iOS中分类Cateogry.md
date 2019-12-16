---
layout: post
title: 详解iOS中分类Cateogry
date: 2017-06-06 15:32:24.000000000 +09:00
categories: 
- iOS
---

首发于[我的个人博客](https://ityongzhen.github.io/%E8%AF%A6%E8%A7%A3iOS%E4%B8%AD%E5%88%86%E7%B1%BBCateogry.html)

## 分类的基本使用

- 首先我们定义一个类 YZPerson 继承自 NSObject

~~~~
@interface YZPerson : NSObject
@end
~~~~

- 然后定义一个分类 YZPerson+test1.h

~~~~
#import "YZPerson.h"

@interface YZPerson (test1)
-(void)run;
@end


#import "YZPerson+test1.h"

@implementation YZPerson (test1)
-(void)run{
    NSLog(@"%s",__func__);
}
@end
~~~~

- 在控制器 ViewController 中使用

~~~~
- (void)viewDidLoad {
    [super viewDidLoad];
    
    YZPerson *person = [[YZPerson alloc] init];
    [person run];
}
~~~~

- 执行结果为


> CateogryDemo[23773:321096] -[YZPerson(test1) run]


**注意点：如果原来的类和分类中有同样的方法，那么执行的结果的是分类中的，例如**

~~~~
#import <Foundation/Foundation.h>

@interface YZPerson : NSObject
-(void)run;
@end



#import "YZPerson.h"

@implementation YZPerson
-(void)run{
    NSLog(@"%s",__func__);
}
@end
~~~~

- 执行结果不会发生变化，依然是

>CateogryDemo[23773:321096] -[YZPerson(test1) run]

原因在后面分析


## 分类的结构

### 打开终端，进入项目下，执行如下命令，生成C语言的文件

> xcrun -sdk iphoneos clang -arch arm64 -rewrite-objc YZPerson+test1.m

### 生成 YZPerson+test1.cpp文件
摘取主要代码如下

~~~~

struct _category_t {
	const char *name;
	struct _class_t *cls;
	const struct _method_list_t *instance_methods;
	const struct _method_list_t *class_methods;
	const struct _protocol_list_t *protocols;
	const struct _prop_list_t *properties;
};

static struct /*_method_list_t*/ {
	unsigned int entsize;  // sizeof(struct _objc_method)
	unsigned int method_count;
	struct _objc_method method_list[1];
} _OBJC_$_CATEGORY_INSTANCE_METHODS_YZPerson_$_test1 __attribute__ ((used, section ("__DATA,__objc_const"))) = {
	sizeof(_objc_method),
	1,
	{{(struct objc_selector *)"run", "v16@0:8", (void *)_I_YZPerson_test1_run}}
};


extern "C" __declspec(dllimport) struct _class_t OBJC_CLASS_$_YZPerson;

static struct _category_t _OBJC_$_CATEGORY_YZPerson_$_test1 __attribute__ ((used, section ("__DATA,__objc_const"))) = 
{
	"YZPerson",
	0, // &OBJC_CLASS_$_YZPerson,
	(const struct _method_list_t *)&_OBJC_$_CATEGORY_INSTANCE_METHODS_YZPerson_$_test1,
	0,
	0,
	0,
};


static void OBJC_CATEGORY_SETUP_$_YZPerson_$_test1(void ) {
	_OBJC_$_CATEGORY_YZPerson_$_test1.cls = &OBJC_CLASS_$_YZPerson;
}

~~~~

说明编译完之后每一个分类都会生成一个 

>_category_t


的结构体，里面有名称，对象方法列表，类方法列表，协议方法列表，属性列表,如果对应的为空，比如协议为空，属性为空，那么结构体中保存的就是0。

### objc-runtime-new.h

打开源码最新的源码 [runtime源码](https://opensource.apple.com/tarballs/objc4/)看，objc-runtime-new.h中分类结构体是这样的

~~~~
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
~~~~


## 源码分析

### 源码解读顺序

~~~~
objc-os.mm
_objc_init
map_images
map_images_nolock

objc-runtime-new.mm
_read_images
remethodizeClass
attachCategories
attachLists
realloc、memmove、 memcpy
~~~~

- 先找到 objc-os.mm 类，里面的

~~~~

// runtime初始化方法
void _objc_init(void) //
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

- 继续跟下去

~~~~
map_images(unsigned count, const char * const paths[],
           const struct mach_header * const mhdrs[])
{
    mutex_locker_t lock(runtimeLock);
    return map_images_nolock(count, paths, mhdrs);
}

~~~~

- 查看 

~~~~
map_images_nolock
~~~~

找到

~~~~
if (hCount > 0) {
        _read_images(hList, hCount, totalClasses, unoptimizedTotalClasses);
    }
~~~~

到了文件 objc-runtime-new.mm 中

~~~~
void _read_images(header_info **hList, uint32_t hCount, int totalClasses, int unoptimizedTotalClasses)
{
	// 关键代码
	 remethodizeClass(cls);
	 // 关键代码
     remethodizeClass(cls->ISA());

}

~~~~

~~~~
static void remethodizeClass(Class cls)
{
    category_list *cats;
    bool isMeta;

    runtimeLock.assertLocked();

    isMeta = cls->isMetaClass();

    // Re-methodizing: check for more categories
    if ((cats = unattachedCategoriesForClass(cls, false/*not realizing*/))) {
        if (PrintConnecting) {
            _objc_inform("CLASS: attaching categories to class '%s' %s", 
                         cls->nameForLogging(), isMeta ? "(meta)" : "");
        }
        
        attachCategories(cls, cats, true /*flush caches*/);        
        free(cats);
    }
}

~~~~

主要代码合注释已经在代码中展示了

~~~~
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    // cats 分类列表
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);
    
    bool isMeta = cls->isMetaClass();
    // 方法数组 二维数组
    // fixme rearrange to remove these intermediate allocations
    method_list_t **mlists = (method_list_t **)
    malloc(cats->count * sizeof(*mlists));
    // 属性数组 二维数组
    property_list_t **proplists = (property_list_t **)
    malloc(cats->count * sizeof(*proplists));
    // 协议数组 二维数组
    protocol_list_t **protolists = (protocol_list_t **)
    malloc(cats->count * sizeof(*protolists));
    
    // Count backwards through cats to get newest categories first
    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    // 这个while循环 合并分类中的 对象方法 属性 协议
    while (i--) {
        // 取出某个分类
        auto& entry = cats->list[i];
        // 取出分类中的对象方法
        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }
        // 取出分类中的属性
        property_list_t *proplist =
        entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }
        // 取出分类中的协议
        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }
    // 得到类对象里面的数据
    auto rw = cls->data();
    
    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    // 所有分类的对象方法，附加到类对象的方法列表中
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);
    // 所有分类的属性，附加到类对象的属性列表中，
    rw->properties.attachLists(proplists, propcount);
    free(proplists);
    // 所有分类的协议，附加到类对象的协议列表中
    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
~~~~

### memmove memcpy

上面的代码继续跟下去来到了

~~~~
void attachLists(List* const * addedLists, uint32_t addedCount) {
        if (addedCount == 0) return;
        
        if (hasArray()) {
            // many lists -> many lists
            uint32_t oldCount = array()->count;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
            array()->count = newCount;
            // 内存挪动
            memmove(array()->lists + addedCount, 
            				array()->lists,
                    oldCount * sizeof(array()->lists[0]));
            // 内存拷贝
            memcpy(array()->lists, addedLists,
                   addedCount * sizeof(array()->lists[0]));
        }
        else if (!list  &&  addedCount == 1) {
            // 0 lists -> 1 list
            list = addedLists[0];
        }
        else {
            // 1 list -> many lists
            List* oldList = list;
            uint32_t oldCount = oldList ? 1 : 0;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)malloc(array_t::byteSize(newCount)));
            array()->count = newCount;
            if (oldList) array()->lists[addedCount] = oldList;
            memcpy(array()->lists, addedLists,
                   addedCount * sizeof(array()->lists[0]));
        }
    }

~~~~

其中关键代码是

~~~~
// 内存挪动
 memmove(array()->lists + addedCount,
                    array()->lists,
                    oldCount * sizeof(array()->lists[0]));
// 内存拷贝
 memcpy(array()->lists, addedLists,
                   addedCount * sizeof(array()->lists[0]));
~~~~

关于memcpy与memmove的区别，可以参考 [memcpy与memmove的区别](https://www.jianshu.com/p/9c3784d8d8ad)

简单总结就是：
> 区别就在于关键字restrict, memcpy假定两块内存区域没有数据重叠，而memmove没有这个前提条件。如果复制的两个区域存在重叠时使用memcpy，其结果是不可预知的，有可能成功也有可能失败的，所以如果使用了memcpy,程序员自身必须确保两块内存没有重叠部分
> 

## 总结
合并分类的时候，其方法列表等，不会覆盖掉原来类中的方法，是共存的。但是分类中的方法在前面，原来的类中的方法在后面，调用的时候，就会调用分类中的方法，如果多个分类有同样的方法，后编译的分类会调用。



## 问题
### Category的使用场合是什么？
- 不同模块的功能区分开来，可以使用分类实现

### Category的实现原理
- Category编译之后的底层结构是struct category_t，里面存储着分类的对象方法、类方法、属性、协议信息
在程序运行的时候，runtime会将Category的数据，合并到类信息中（类对象、元类对象中）

### Category和Class Extension的区别是什么？
Class Extension在编译的时候，它的数据就已经包含在类信息中
Category是在运行时，才会将数据合并到类信息中

### Category中有load方法吗？load方法是什么时候调用的？load 方法能继承吗？
有load方法
load方法在runtime加载类、分类的时候调用
load方法可以继承，但是一般情况下不会主动去调用load方法，都是让系统自动调用

### Category能否添加成员变量？如果可以，如何给Category添加成员变量？
- 不能直接给Category添加成员变量，但是可以间接实现Category有成员变量的效果,关联对象










本文相关代码github地址 [github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos)

本文参考资料:

[runtime源码](https://opensource.apple.com/tarballs/objc4/)

[iOS底层原理](https://ke.qq.com/course/package/11609)

[memcpy与memmove的区别](https://www.jianshu.com/p/9c3784d8d8ad)






