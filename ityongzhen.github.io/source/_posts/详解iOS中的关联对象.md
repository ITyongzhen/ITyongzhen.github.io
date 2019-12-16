---
layout: post
title: 详解iOS中的关联对象
date: 2017-07-06 15:32:24.000000000 +09:00
categories: 
- iOS
---


首发于[我的个人博客](https://ityongzhen.github.io/详解iOS中的关联对象.html)

## 从给分类添加属性说起


在[详解iOS中分类Cateogry](https://juejin.im/post/5d204b77f265da1bcc196313) 一文中，我们提出一个问题，

### Category能否添加成员变量？如果可以，如何给Category添加成员变量？

- 不能直接给Category添加成员变量，但是可以间接实现Category有成员变量的效果,用关联对象技术

那这里就详细说明

### 添加属性，实际上都做了什么

首先我们要回忆一下，添加属性，实际上做了三件事

- 生成成员变量
- 生成set方法和get方法的声明
- 生成set方法和get方法的实现

eg：
定义一个 `YZPerson` 类，并定义`age`属性

~~~~
#import <Foundation/Foundation.h>

@interface YZPerson : NSObject

@property (assign, nonatomic) int age;


@end

~~~~

就相当于干了三件事

- 生成成员变量`_age `
- 生成`set`方法和`get`方法的声明
- 生成`set`方法和`get`方法的实现
如下

~~~~
#import <Foundation/Foundation.h>


@interface YZPerson : NSObject

{
    int _age;
}
- (void)setAge:(int)age;
- (int)age;

@end



#import "YZPerson.h"

@implementation YZPerson
- (void)setAge:(int)age{
    _age = age;
}

- (int)age{
    return _age;
}
@end

~~~~


### 那在分类中添加属性怎么就不行？
#### 先说结论

- 生成成员变量`_age `
- 不会生成`set`方法和`get`方法的声明
- 不会生成`set`方法和`get`方法的实现

#### 不会生成`set`方法和`get`方法的实现

定义一个分类 `YZPerson+Ext.h`，然后添加属性`weight`

~~~~
#import "YZPerson.h"
@interface YZPerson (Ext)
@property (nonatomic ,assign)  int weight;
@end
~~~~

使用

~~~~
YZPerson *person = [[YZPerson alloc] init];
person.weight = 10;
~~~~

会直接报错，

~~~~
iOS-关联对象[1009:10944] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', 
reason: '-[YZPerson setWeight:]: unrecognized selector sent to instance 0x10182bd10'
*** First throw call stack:
(
	0   CoreFoundation                      0x00007fff3550d063 __exceptionPreprocess + 250
	1   libobjc.A.dylib                     0x00007fff6ac8e06b objc_exception_throw + 48
	2   CoreFoundation                      0x00007fff355961bd -[NSObject(NSObject) __retain_OA] + 0
	3   CoreFoundation                      0x00007fff354b34b4 ___forwarding___ + 1427
	4   CoreFoundation                      0x00007fff354b2e98 _CF_forwarding_prep_0 + 120
	
	6   libdyld.dylib                       0x00007fff6c0183f9 start + 1
)
libc++abi.dylib: terminating with uncaught exception of type NSException
Program ended with exit code: 9
~~~~

从 `reason: '-[YZPerson setWeight:]: unrecognized selector sent to instance 0x10182bd10'` 可知，分类中添加属性，没有生成`set`方法和`get`方法的实现

#### 会生成`set`方法和`get`方法的声明


~~~~
#import "YZPerson+Ext.h"

@implementation YZPerson (Ext)
- (void)setWeight:(int)weight{
    
}
- (int)weight{
    return 100;
}
@end

~~~~

然后再调用

~~~~
YZPerson *person = [[YZPerson alloc] init];
person.age = 25;
person.weight = 10;
NSLog(@"person.age = %d",person.age);
NSLog(@"person.weight = %d",person.weight);
~~~~

输出

~~~~
2019-07-10 08:28:04.406972+0800 iOS-关联对象[1620:18520] person.age = 25
2019-07-10 08:28:04.407291+0800 iOS-关联对象[1620:18520] person.weight = 100
~~~~

进一步证明了，不会生成`set`方法和`get`方法的实现，但是会生成`set`方法和`get`方法的声明，因为如果没有生成`set`方法和`get`方法的声明，这个方法就不能调用。

我们还可以这样：在`YZPerson+Ext.h`文件中声明了`weight`,然后再`YZPerson+Ext.m`中写实现的时候，会有提示的

![](https://user-gold-cdn.xitu.io/2019/7/11/16bde91859761ab0?w=1068&h=284&f=png&s=157470)

更加说明了是有声明的。

#### 分类中不能直接定义成员变量

~~~~
#import "YZPerson.h"


@interface YZPerson (Ext)
{
    int _weight; // 报错 Instance variables may not be placed in categories
}
@property (nonatomic ,assign)  int weight;
@end

~~~~

会直接报错`Instance variables may not be placed in categories`,成员变量不能定义在分类中

#### 源码角度证明

前面的文章[详解iOS中分类Cateogry](https://ityongzhen.github.io/%E8%AF%A6%E8%A7%A3iOS%E4%B8%AD%E5%88%86%E7%B1%BBCateogry.html) 中分析过源码，objc-runtime-new.h中分类结构体是这样的

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

可知，这个结构体中，没有数组存放成员变量，只有属性，协议等。

## 怎么来完善属性

有什么办法可以实现在分类中添加属性和在类中添加属性一样的效果么？答案是有的

### 方案一 用全局变量

分类`YZPerson+Ext.m`中定义全局变量 `_weight`

~~~~

#import "YZPerson+Ext.h"

@implementation YZPerson (Ext)

int _weight;

- (void)setWeight:(int)weight{
    _weight = weight;
}
- (int)weight{
    return _weight;
}
@end

~~~~

使用时候

~~~~
YZPerson *person = [[YZPerson alloc] init];
person.weight = 103;
NSLog(@"person.weight = %d",person.weight);
~~~~


输出为

~~~~
iOS-关联对象[1983:23793] person.weight = 103
~~~~

看起来确实可以，然后实际上我们不能这么用，因为，全局变量是共享的，假设有两个 `Person`,第二个`Person`修改了weight属性，然后打印第一个`Person.weight`

~~~~
YZPerson *person = [[YZPerson alloc] init];
person.weight = 103;
NSLog(@"person.weight = %d",person.weight);

YZPerson *person2 = [[YZPerson alloc] init];
person2.weight = 10;
NSLog(@"person.weight = %d",person.weight);
~~~~

输出为

~~~~
iOS-关联对象[1983:23793] person.weight = 103
iOS-关联对象[1983:23793] person.weight = 10
~~~~

可知，修改了`Person2.weight` 会改变`Person.weight`的值，因为是全局变量的缘故。所以这种方法不行

### 方案二 用字典

既然前面方案不能用的原因是全局变量，共享一份，那我们是不是只要保证，一对一的关系，是不是就可以了呢？

定义 字典`weights_` 以对象的地址值作为`key`来，`weight`的值作为`value`来存储和使用

~~~~

#import "YZPerson+Ext.h"

@implementation YZPerson (Ext)

NSMutableDictionary *weights_;

+ (void)load{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        // 写在这里，保证s只初始化一次
        weights_ = [NSMutableDictionary dictionary];
    });
}

- (void)setWeight:(int)weight{
    NSString *key = [NSString stringWithFormat:@"%p",self];//self 地址值作为key
    weights_[key] = @(weight);//字典中的value不能直接放int，需要包装成对象
}
- (int)weight{
     NSString *key = [NSString stringWithFormat:@"%p",self];
    return  [weights_[key] intValue];
}

@end
~~~~

![](https://user-gold-cdn.xitu.io/2019/7/11/16bde9185962846f?w=1159&h=553&f=png&s=304359)

这样的话，使用起来，就不会因为不同对象而干扰了
结果如下

![](https://user-gold-cdn.xitu.io/2019/7/11/16bde9185995ab92?w=755&h=532&f=png&s=114477)


#### 存在的问题

- 因为是全局的，存在内存泄露问题
- 线程安全问题，多个线程同时访问的话，有线程安全问题
- 代码太多，如果每次增加一个属性，都要写好多代码。不利于维护

## 关联对象方案
### 关联对象的使用
下面先简单说明关联对象的使用
### 动态添加

~~~~
objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy)
~~~~

- 参数一：`id object` : 给哪个对象添加属性，这里要给自己添加属性，用`self`。
- 参数二：`void * == id key` : `key`值，根据key获取关联对象的属性的值，在`objc_getAssociatedObject`中通过次`key`获得属性的值并返回。
- 参数三：`id value` : 关联的值，也就是`set`方法传入的值给属性去保存。
- 参数四：`objc_AssociationPolicy policy `: 策略，属性以什么形式保存。

~~~~
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,  // 指定一个弱引用相关联的对象
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, // 指定相关对象的强引用，非原子性
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,  // 指定相关的对象被复制，非原子性
    OBJC_ASSOCIATION_RETAIN = 01401,  // 指定相关对象的强引用，原子性
    OBJC_ASSOCIATION_COPY = 01403     // 指定相关的对象被复制，原子性   
};
~~~~

整理成表格如下

| objc_AssociationPolicy  | 对应的修饰符 |
| :--- | :----: | 
| OBJC_ASSOCIATION_ASSIGN | assign |
| OBJC_ASSOCIATION_RETAIN_NONATOMIC |strong, nonatomic |
| OBJC_ASSOCIATION_COPY_NONATOMIC | copy, nonatomic|
| OBJC_ASSOCIATION_RETAIN |strong, atomic |
| OBJC_ASSOCIATION_COPY |copy, atomic |
	
eg: 我们在代码中使用了 `OBJC_ASSOCIATION_RETAIN_NONATOMIC` 就相当于使用了 `nonatomic` 和 `strong` 修饰符。
	
**注意点**
上面列表中，没有对应`weak`修饰的策略，
原因是
`object`经过`DISGUISE`函数被转化为了`disguised_ptr_t`类型的`disguised_object`。

~~~~
disguised_ptr_t disguised_object = DISGUISE(object);
~~~~
而`weak`修饰的属性，当没有拥有对象之后就会被销毁，并且指针置为`nil`，那么在对象销毁之后，虽然在`map`中仍然存在值`object`对应的`AssociationsHashMap`，但是因为`object`地址已经被置为`nil`，会造成坏地址访问而无法根据`object`对象的地址转化为`disguised_object`了,这段话可以再看完全文之后，再回来体会下。



### 取值

~~~~
objc_getAssociatedObject(id object, const void *key);
~~~~

- 参数一：`id object` : 获取哪个对象里面的关联的属性。 
- 参数二：`void * == id key` : 什么属性，与`objc_setAssociatedObject`中的`key`相对应，即通过`key`值取出`value`。

### 移除关联对象

~~~~
- (void)removeAssociatedObjects
{
    // 移除关联对象
    objc_removeAssociatedObjects(self);
}
~~~~

### 具体应用

~~~~

#import "YZPerson.h"

@interface YZPerson (Ext)
@property (nonatomic,strong) NSString *name;
@end


#import "YZPerson+Ext.h"
#import <objc/runtime.h>
@implementation YZPerson (Ext)

const void *YZNameKey = &YZNameKey;

- (void)setName:(NSString *)name{
    objc_setAssociatedObject(self, YZNameKey, name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)name{
   return objc_getAssociatedObject(self, YZNameKey);
}

- (void)dealloc
{
    objc_removeAssociatedObjects(self);
}

@end
~~~~

使用的时候，正常使用，就可以了

~~~~
YZPerson *person = [[YZPerson alloc] init];
person.name = @"jack";

YZPerson *person2 = [[YZPerson alloc] init];
person2.name = @"rose";
        
NSLog(@"person.name = %@",person.name);
NSLog(@"person2.name = %@",person2.name);
~~~~


输出

~~~~
iOS-关联对象[4266:52285] person.name = jack
iOS-关联对象[4266:52285] person2.name = rose
~~~~

使用起来就是这么简单

## 关联对象原理

### 四个核心对象

实现关联对象技术的核心对象有
- AssociationsManager
- AssociationsHashMap
- ObjectAssociationMap
- ObjcAssociation

### 源码解读

关联对象的源码在 [Runtime源码](https://opensource.apple.com/tarballs/objc4/)中


#### `objc_setAssociatedObject`

查看`objc-runtime.mm`类，首先找到`objc_setAssociatedObject`函数，看一下其实现

~~~~
void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy) {
    _object_set_associative_reference(object, (void *)key, value, policy);
}
~~~~

#### `_object_set_associative_reference`
查看

~~~~

void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    // retain the new value (if any) outside the lock.
    ObjcAssociation old_association(0, nil);
    id new_value = value ? acquireValue(value, policy) : nil;
    {
    	
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {
            // break any existing association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } else {
            // setting the association to nil breaks the association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    refs->erase(j);
                }
            }
        }
    }
    // release the old value (outside of the lock).
    if (old_association.hasValue()) ReleaseValue()(old_association);
}

~~~~

如图所示

![](https://user-gold-cdn.xitu.io/2019/7/11/16bde9185a849239?w=2174&h=1462&f=png&s=415822)

`_object_set_associative_reference`函数内部我们可以找到我们上面说过的实现关联对象技术的四个核心对象。接下来我们来一个一个看其内部实现原理探寻他们之间的关系。

#### `AssociationsManager`

查看 `AssociationsManager` 我们知道`AssociationsManager` 内部有` static AssociationsHashMap *_map;`


~~~~

class AssociationsManager {
    // associative references: object pointer -> PtrPtrHashMap.
    static AssociationsHashMap *_map;
public:
    AssociationsManager()   { AssociationsManagerLock.lock(); }
    ~AssociationsManager()  { AssociationsManagerLock.unlock(); }
    
    AssociationsHashMap &associations() {
        if (_map == NULL)
            _map = new AssociationsHashMap();
        return *_map;
    }
};
~~~~

#### `AssociationsHashMap`
接下来看 `AssociationsHashMap`		
![](https://user-gold-cdn.xitu.io/2019/7/11/16bde9185a9ae0f4?w=2366&h=1438&f=png&s=469605)


上图中 `AssociationsHashMap`的源码我们发现`AssociationsHashMap`继承自`unordered_map`首先来看一下`unordered_map`内的源码

![](https://user-gold-cdn.xitu.io/2019/7/11/16bde9185d81f1f1?w=2120&h=986&f=png&s=342730)

从`unordered_map`源码中我们可以看出 参数 `_Key`和`_Tp` 对应着`map`中的`Key`和`Value`，那么对照上面`AssociationsHashMap`的源码，可以发现`_Key`中传入的是`unordered_map<disguised_ptr_t`，`_Tp`中传入的值则为`ObjectAssociationMap *`。

然后 我们查看`ObjectAssociationMap`的源码，上图中`ObjectAssociationMap`已经标记出，我们可以知道`ObjectAssociationMap`中同样以`key`、`Value`的方式存储着`ObjcAssociation`。

#### `ObjcAssociation `
接着我们来到`ObjcAssociation`中，可以看到

~~~~
 class ObjcAssociation {
        uintptr_t _policy; // 策略
        id _value; // value值
    public:
        ObjcAssociation(uintptr_t policy, id value) : _policy(policy), _value(value) {}
        ObjcAssociation() : _policy(0), _value(nil) {}

        uintptr_t policy() const { return _policy; }
        id value() const { return _value; }
        
        bool hasValue() { return _value != nil; }
    };
~~~~

从上面的代码中，我们发现`ObjcAssociation`存储着`_policy`和`_value`，而这两个值我们可以发现正是我们调用`objc_setAssociatedObject`函数传入的值，换句话说我们在调用`objc_setAssociatedObject`函数中传入`value`和`policy`这两个值最终是存储在`ObjcAssociation`中的。

现在我们已经对四个核心对象`AssociationsManager`、 `AssociationsHashMap`、 `ObjectAssociationMap`、`ObjcAssociation`之间的关系有了初步的了解，那么接下继续仔细阅读源码，看一下`objc_setAssociatedObject`函数中传入的四个参数分别放在哪个对象中充当什么作用

#### 细读 `_object_set_associative_reference `

`_object_set_associative_reference `的代码中

~~~~


void _object_set_associative_reference(id object, void *key, id value, uintptr_t policy) {
    // retain the new value (if any) outside the lock.
    ObjcAssociation old_association(0, nil);
    // 根据value的值通过acquireValue函数获取得到new_value
    id new_value = value ? acquireValue(value, policy) : nil;
    {
        AssociationsManager manager;
        // 获取 manager 内的 AssociationsHashMap 也就是 associations
        AssociationsHashMap &associations(manager.associations());
        // object 经过 DISGUISE 函数被转化为了disguised_ptr_t类型的disguised_object
        disguised_ptr_t disguised_object = DISGUISE(object);
        if (new_value) {
            // break any existing association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i != associations.end()) {
                // secondary table exists
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    // policy和new_value 作为键值对存入了ObjcAssociation
                    j->second = ObjcAssociation(policy, new_value);
                } else {
                    // policy和new_value 作为键值对存入了ObjcAssociation
                    (*refs)[key] = ObjcAssociation(policy, new_value);
                }
            } else {
                // create the new association (first time).
                ObjectAssociationMap *refs = new ObjectAssociationMap;
                associations[disguised_object] = refs;
                (*refs)[key] = ObjcAssociation(policy, new_value);
                object->setHasAssociatedObjects();
            }
        } else {
            // 来到这里说明，value为空
            // setting the association to nil breaks the association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    //移除关联对象
                    refs->erase(j);
                }
            }
        }
    }
    // release the old value (outside of the lock).
    if (old_association.hasValue()) ReleaseValue()(old_association);
}
~~~~




**`acquireValue `**内部实现 通过对策略的判断返回不同的值

~~~~
static id acquireValue(id value, uintptr_t policy) {
    switch (policy & 0xFF) {
    case OBJC_ASSOCIATION_SETTER_RETAIN:
        return objc_retain(value);
    case OBJC_ASSOCIATION_SETTER_COPY:
        return ((id(*)(id, SEL))objc_msgSend)(value, SEL_copy);
    }
    return value;
}
~~~~

- 首先根据我们传入的`value`经过`acquireValue`函数处理返回了`new_value`。`acquireValue`函数内部其实是通过对策略的判断返回不同的值

~~~~
 typedef uintptr_t disguised_ptr_t;
    inline disguised_ptr_t DISGUISE(id value) { return ~uintptr_t(value); }
    inline id UNDISGUISE(disguised_ptr_t dptr) { return id(~dptr); }
~~~~

- 之后创建`AssociationsManager manager`,得到`manager`内部的`AssociationsHashMap`即`associations`。
之后我们看到了我们传入的第一个参数`object`经过`DISGUISE`函数被转化为了`disguised_ptr_t`类型的`disguised_object`。

~~~~
typedef uintptr_t disguised_ptr_t;
inline disguised_ptr_t DISGUISE(id value) { return ~uintptr_t(value); }
inline id UNDISGUISE(disguised_ptr_t dptr) { return id(~dptr); }
~~~~

- 之后被处理成`new_value`的`value`，和`policy`一起被存入了`ObjcAssociation`中。
而`ObjcAssociation`对应我们传入的`key`被存入了`ObjectAssociationMap`中。
`disguised_object`和`ObjectAssociationMap`则以`key-value`的形式对应存储在`associations`中也就是`AssociationsHashMap`中。

#### value为空

如果传入的value为空，那么就删除这个关联对象

~~~~
    // 来到这里说明，value为空
            // setting the association to nil breaks the association.
            AssociationsHashMap::iterator i = associations.find(disguised_object);
            if (i !=  associations.end()) {
                ObjectAssociationMap *refs = i->second;
                ObjectAssociationMap::iterator j = refs->find(key);
                if (j != refs->end()) {
                    old_association = j->second;
                    //移除关联对象
                    refs->erase(j);
                }
            }
~~~~

本文参考资料:

#### 表格总结
用表格总结来展示这几个核心类的关系如下

![](https://user-gold-cdn.xitu.io/2019/7/11/16bde918b73b14ee?w=851&h=408&f=png&s=78237)

#### 小结

- 关联对象并不存储在被关联对象本身内存中，而是有一个全局统一的 `AssociationsManager`中
- 一个实例对象就对应一个`ObjectAssociationMap`，
- 而`ObjectAssociationMap`中存储着多个此实例对象的关联对象的`key`以及`ObjcAssociation`，
- `ObjcAssociation`中存储着关联对象的`value`和`policy`策略

#### **objc_getAssociatedObject**
`objc_getAssociatedObject`内部调用的是`_object_get_associative_reference`

~~~~
id objc_getAssociatedObject(id object, const void *key) {
    return _object_get_associative_reference(object, (void *)key);
}
~~~~

#### `_object_get_associative_reference`函数


~~~~
id _object_get_associative_reference(id object, void *key) {
    id value = nil;
    uintptr_t policy = OBJC_ASSOCIATION_ASSIGN;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        disguised_ptr_t disguised_object = DISGUISE(object);
        // 查找 disguised_object
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) {
            ObjectAssociationMap *refs = i->second;
            //查看key 和value
            ObjectAssociationMap::iterator j = refs->find(key);
            if (j != refs->end()) {
                ObjcAssociation &entry = j->second;
                value = entry.value();
                policy = entry.policy();
                // 存在key 和value 就取出对应的值
                if (policy & OBJC_ASSOCIATION_GETTER_RETAIN) {
                    objc_retain(value);
                }
            }
        }
    }
    if (value && (policy & OBJC_ASSOCIATION_GETTER_AUTORELEASE)) {
        // 不存在key value 就把这个关联对象擦除
        objc_autorelease(value);
    }
    return value;
}
~~~~

关键代码已经在上文中给了注释

#### `objc_removeAssociatedObjects`函数

`objc_removeAssociatedObjects`函数用来删除所有关联对象，内部调用了`_object_remove_assocations`

~~~~
void objc_removeAssociatedObjects(id object) 
{
    if (object && object->hasAssociatedObjects()) {
        _object_remove_assocations(object);
    }
}
~~~~

#### `_object_remove_assocations`
再来看看`_object_remove_assocations`

~~~~
void _object_remove_assocations(id object) {
    vector< ObjcAssociation,ObjcAllocator<ObjcAssociation> > elements;
    {
        AssociationsManager manager;
        AssociationsHashMap &associations(manager.associations());
        if (associations.size() == 0) return;
        disguised_ptr_t disguised_object = DISGUISE(object);
        AssociationsHashMap::iterator i = associations.find(disguised_object);
        if (i != associations.end()) { // 遍历AssociationsHashMap 取出值
            // copy all of the associations that need to be removed.
            ObjectAssociationMap *refs = i->second;
            for (ObjectAssociationMap::iterator j = refs->begin(), end = refs->end(); j != end; ++j) {
                elements.push_back(j->second);
            }
            // remove the secondary table.
            delete refs;
            // 删除
            associations.erase(i);
        }
    }
    // the calls to releaseValue() happen outside of the lock.
    for_each(elements.begin(), elements.end(), ReleaseValue());
}
~~~~

代码中可以看出，接受一个`object`对象，然后遍历删除该对象所有的关联对象


## 总结
用表格总结来展示这几个核心类的关系如下

![](https://user-gold-cdn.xitu.io/2019/7/11/16bde918b73b14ee?w=851&h=408&f=png&s=78237)



- 关联对象并不存储在被关联对象本身内存中，而是有一个全局统一的 `AssociationsManager`中
- 一个实例对象就对应一个`ObjectAssociationMap`，
- 而`ObjectAssociationMap`中存储着多个此实例对象的关联对象的`key`以及`ObjcAssociation`，
- `ObjcAssociation`中存储着关联对象的`value`和`policy`策略
- 删除的时候接收一个`object`对象，然后遍历删除该对象所有的关联对象
- 设置关联对象`_object_set_associative_reference`的是时候，如果传入的`value`为空就删除这个关联对象


本文参考资料:

本文相关代码github地址 [github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos)

[Runtime源码](https://opensource.apple.com/tarballs/objc4/)

[iOS底层原理](https://ke.qq.com/course/package/11609)

