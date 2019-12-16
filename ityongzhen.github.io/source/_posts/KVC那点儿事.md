---
layout: post
title: KVC那点儿事
date: 2018-05-15 19:32:24.000000000 +09:00
categories: 
- iOS
---
本文首发于[个人博客](https://ityongzhen.github.io/KVC%E9%82%A3%E7%82%B9%E5%84%BF%E4%BA%8B.html)
## 前言
- KVC是Key Value Coding的简称。它是一种可以通过字符串的名字（key）来访问类属性的机制。而不是通过调用Setter、Getter方法访问。KVC的方法定义在Foundation/NSKeyValueCoding中。
- KVC和KVO都属于键值编程而且底层实现机制都是**isa-swizzing**。

常见的API有

- -(void)setValue:(id)value forKeyPath:(NSString *)keyPath;
- -(void)setValue:(id)value forKey:(NSString *)key;
- -(id)valueForKeyPath:(NSString *)keyPath;
- -(id)valueForKey:(NSString *)key; 



## KVC基本使用
- 定义一个YZPerson类，有个 name 属性

~~~~
@interface YZPerson : NSObject
@property (nonatomic,strong) NSString *name;
@end
~~~~

- ViewController 控制器中，如下使用

~~~~
#import "ViewController.h"
#import "YZPerson.h"
@interface ViewController ()

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    YZPerson *person = [[YZPerson alloc]init];
     // 赋值
    [person setValue:@"jack" forKey:@"name"];
    // 取值
    NSLog(@"%@",[person valueForKey:@"name"]);
}


@end
~~~~

- 结果为

~~~~
KVCDemo[25838:347883] jack
~~~~

## 赋值  setValue:forKey:的原理

1. 按照 **setKey:**、**_setKey:** 的顺序查找方法
2. 如果找到了方法，就传递参数，调用方法
3. 如果没有找到，查看 **accessInstanceVariablesDirectly** 方法的返回值
4. 如果**accessInstanceVariablesDirectly** 返回值为 **NO** 调用
**setValue:forUndefinedKey:** 并抛出异常 **NSUnknownKeyException**
5. 如果**accessInstanceVariablesDirectly** 返回值为 **YES** 按照**_key**、**_isKey**、**key**、**isKey**的顺序查找成员变量
6. 如果找到了成员变量，就直接赋值。
7. 如果 **_key**、**_isKey**、**key**、**isKey**的顺序没有查找到成员变量就调用**setValue:forUndefinedKey:** 并抛出异常 **NSUnknownKeyException**

### 证明赋值

#### 先证明 按照 **setKey:**、**_setKey:** 的顺序查找方法

**YZPerson.h** 和 **YZPerson.m** 如下

~~~~

// 只有name属性，没有age
@interface YZPerson : NSObject
@property (nonatomic,strong) NSString *name;
@end

@implementation YZPerson
- (void)setAge:(int)age
{
    NSLog(@"setAge: - %d", age);
}

- (void)_setAge:(int)age
{
    NSLog(@"_setAge: - %d", age);
}

~~~~

调用地方 

~~~~
 YZPerson *person = [[YZPerson alloc]init];
    // 赋值
 [person setValue:@20 forKey:@"age"];

~~~~

打印结果是

~~~~
KVCDemo[26389:357519] setAge: - 20
~~~~

说明调用来的是**setAge:** 那如果 去掉 **setAge:** 呢

~~~~

// 只有name属性，没有age
@interface YZPerson : NSObject
@property (nonatomic,strong) NSString *name;
@end

@implementation YZPerson
- (void)setAge:(int)age
{
    NSLog(@"setAge: - %d", age);
}

- (void)_setAge:(int)age
{
    NSLog(@"_setAge: - %d", age);
}

~~~~

结果为:

~~~~
KVCDemo[26594:360894] _setAge: - 20
~~~~

证明了 按照 **setKey:**、**_setKey:** 的顺序查找方法

### 证明 accessInstanceVariablesDirectly
1. 如果**accessInstanceVariablesDirectly** 返回值为 **NO** 调用**setValue:forUndefinedKey:** 并抛出异常 **NSUnknownKeyException**
2. 如果**accessInstanceVariablesDirectly** 返回值为 **YES** 就去查找成员变量，就直接赋值。

- 我们在 YZPerson.h 中定义四个成员变量, YZPerson.m中 只有accessInstanceVariablesDirectly 并返回NO

~~~~

@interface YZPerson : NSObject
{
@public
        int age;
        int isAge;
        int _isAge;
        int _age;
}
@property (nonatomic,strong) NSString *name;
@end


#import "YZPerson.h"

@implementation YZPerson

// 默认的返回值就是YES
+ (BOOL)accessInstanceVariablesDirectly
{
    return NO;
}
@end

~~~~

运行报错:找不到 key值 age

~~~~
 KVCDemo[27163:369895] *** Terminating app due to uncaught exception 
 'NSUnknownKeyException', reason: '[<YZPerson 0x600003a5e700> 
 setValue:forUndefinedKey:]: this class is not key value 
 coding-compliant for the key age.'
~~~~
 
- 我们把YZPerson.m中 只有accessInstanceVariablesDirectly 返回YES

运行结果：

~~~~
KVCDemo[27385:373752] 20
~~~~

### 证明是按照**_key**、**_isKey**、**key**、**isKey**的顺序查找成员变量
 
代码还是上面的代码，打断点，然后LLDB调试

~~~~
(lldb) po person->_age
20

(lldb) po person->_isAge
<nil>

(lldb) po person->age
<nil>

(lldb) po person->isAge
<nil>

~~~~
 
如果去掉成员变量**_age**

结果为

~~~~

(lldb) po person->_isAge
20

(lldb) po person->age
<nil>

(lldb) po person->isAge
<nil>

~~~~

同理其他的几种情况，读者可自行尝试 [demo]([github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos))。

## KVC与KVO

通过[关于KVO看这篇就够了](https://ityongzhen.github.io/%E5%85%B3%E4%BA%8EKVO%E7%9C%8B%E8%BF%99%E7%AF%87%E5%B0%B1%E5%A4%9F%E4%BA%86.html) 我们知道

### KVO的本质

- 	利用RuntimeAPI动态生成一个子类，并且让instance对象的isa指向这个全新的子类
-  当修改instance对象的属性时，会调用Foundation的_NSSetXXXValueAndNotify函数
	- willChangeValueForKey:
	- 父类原来的setter
	- didChangeValueForKey:
- 内部会触发监听器（Oberser）的监听方法( observeValueForKeyPath:ofObject:change:context:）

那么KVC能否触发KVO呢，

我们在 YZPerson.h中书写如下代码

~~~~
@interface YZPerson : NSObject
{
@public
        int age;
        int isAge;
        int _isAge;
        int _age;
}
@property (nonatomic,strong) NSString *name;
@end
~~~~

我们知道，成员变量是不会生成set 和 get方法的
然后 YZPerson.m中书写如下代码

~~~~
#import "YZPerson.h"

@implementation YZPerson

// 默认的返回值就是YES
+ (BOOL)accessInstanceVariablesDirectly
{
    return YES;
}
@end

~~~~

在VC中设置KVO监听

~~~~

#import "ViewController.h"
#import "YZPerson.h"
@interface ViewController ()
@property (nonatomic,strong) YZPerson *person;
@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.person = [[YZPerson alloc]init];
    
    // 添加KVO监听
    [self.person addObserver:self forKeyPath:@"age" options:NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld context:NULL];
    
    // 通过KVC修改age属性
    [self.person setValue:@10 forKey:@"age"];

    // 取值
    NSLog(@"取值为：%@",[self.person valueForKey:@"age"]);
}

- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context
{
    NSLog(@"observeValueForKeyPath - %@", change);
}

-(void)dealloc{
    // 移除KVO监听
    [self.person removeObserver:self forKeyPath:@"age"];
}

~~~~

输出结果为：

~~~~
KVCDemo[28271:388786] observeValueForKeyPath - {
    kind = 1;
    new = 10;
    old = 0;
}
KVCDemo[28271:388786] 取值为：10
~~~~

可知，其实在系统内部，是调用了 

	- willChangeValueForKey:
	
	- didChangeValueForKey:

### 进一步验证

然后 YZPerson.m中书写如下代码

~~~~
#import "YZPerson.h"

@implementation YZPerson

- (void)willChangeValueForKey:(NSString *)key
{
    [super willChangeValueForKey:key];
    NSLog(@"willChangeValueForKey - %@", key);
}

- (void)didChangeValueForKey:(NSString *)key
{
    NSLog(@"didChangeValueForKey - begin - %@", key);
    [super didChangeValueForKey:key];
    NSLog(@"didChangeValueForKey - end - %@", key);
}
// 默认的返回值就是YES
+ (BOOL)accessInstanceVariablesDirectly
{
    return YES;
}
@end

~~~~

输出结果为：

~~~~

KVCDemo[28392:390730] willChangeValueForKey - age
KVCDemo[28392:390730] didChangeValueForKey - begin - age
KVCDemo[28392:390730] observeValueForKeyPath - {
    kind = 1;
    new = 10;
    old = 0;
}
KVCDemo[28392:390730] didChangeValueForKey - end - age
KVCDemo[28392:390730] 取值为：10
~~~~

所以，足以说明，KVC内部调用了 **willChangeValueForKey** 和 **didChangeValueForKey**

## 取值  valueForKey:的原理

1. 按照**getKey**、**key**、**isKey**、**_key**的顺序查找方法
2. 如果找到了，就直接调用
3. 如果没找到，就查看**accessInstanceVariablesDirectly** 方法的返回值
4. 如果**accessInstanceVariablesDirectly** 返回值为 **NO** 调用**valueForUndefinedKey:**并抛出异常**NSUnknownKeyException**
5. 如果**accessInstanceVariablesDirectly** 返回值为 **YES** 按照**_key**、**_isKey**、**key**、**isKey**的顺序查找成员变量
6. 如果找到了成员变量，就直接取值。
7. 如果 **_key**、**_isKey**、**key**、**isKey**的顺序没有查找到成员变量就调用**valueForUndefinedKey:**并抛出异常**NSUnknownKeyException**


### 验证取值

- 取值和赋值的大体逻辑基本一致

然后 YZPerson.m中书写如下代码

~~~~
#import "YZPerson.h"

@implementation YZPerson

- (int)getAge
{
    return 11;
}

- (int)age
{
    return 12;
}

- (int)isAge
{
    return 13;
}

- (int)_age
{
    return 14;
}
// 默认的返回值就是YES
+ (BOOL)accessInstanceVariablesDirectly
{
    return YES;
}
@end

~~~~

VC中如下代码

~~~~
- (void)viewDidLoad {
    [super viewDidLoad];
    
    self.person = [[YZPerson alloc]init];

    // 取值
    NSLog(@"取值为：%@",[self.person valueForKey:@"age"]);
}


~~~~

结果为

~~~~
KVCDemo[29145:403008] 取值为：11
~~~~

如果去掉  

~~~~
- (int)getAge
{
    return 11;
}
~~~~

则，输出结果为12。

上面验证了 按照**getKey**、**key**、**isKey**、**_key**的顺序查找方法


其他的验证逻辑，和赋值验证过程一致，就不赘述了。



本文相关代码github地址 [github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos)

本文参考资料:


[Runtime源码](https://opensource.apple.com/tarballs/objc4/)

[iOS底层原理](https://ke.qq.com/course/package/11609)






