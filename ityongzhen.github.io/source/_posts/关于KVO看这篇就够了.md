---
layout: post
title: 关于KVO看这篇就够了
date: 2017-05-26 19:32:24.000000000 +09:00
categories: 
- iOS
---

本文首发于[个人博客](https://ityongzhen.github.io/%E5%85%B3%E4%BA%8EKVO%E7%9C%8B%E8%BF%99%E7%AF%87%E5%B0%B1%E5%A4%9F%E4%BA%86.html)

## 前言
- KVO全称KeyValueObserving，俗称**键值监听**，是苹果提供的一套事件通知机制。允许对象监听另一个对象特定属性的改变，并在改变时接收到事件。由于KVO的实现机制，所以对属性才会发生作用，一般继承自NSObject的对象都默认支持KVO。
- KVC和KVO都属于键值编程而且底层实现机制都是**isa-swizzing**。
- KVO和NSNotificationCenter都是iOS中**观察者模式**的一种实现。KVO对被监听对象无侵入性，不需要修改其内部代码即可实现监听。
- KVO可以监听单个属性的变化，也可以监听集合对象的变化。通过KVC的mutableArrayValueForKey:等方法获得代理对象，当代理对象的内部对象发生改变时，会回调KVO监听的方法。集合对象包含NSArray和NSSet。

## 实现原理

- KVO是通过isa-swizzling技术实现的(这句话是整个KVO实现的重点)。
- 在运行时根据原类创建一个中间类，这个中间类是原类的子类，并动态修改当前对象的isa指向中间类。当修改 instance 对象的属性时，会调用 Foundation框架的 _NSSetXXXValueAndNotify 函数 ,该函数里面会先调用 willChangeValueForKey: 然后调用父类原来的 setter 方法修改值，最后是 didChangeValueForKey:。didChangeValueForKey 内部会触发监听器（Oberser）的监听方法observeValueForKeyPath:ofObject:change:context:
- 并且将class方法重写，返回原类的Class。




## KVO的使用
### 使用方法
1. 通过addObserver:forKeyPath:options:context:方法注册观察者，观察者可以接收keyPath属性的变化事件。
2. 在观察者中实现observeValueForKeyPath:ofObject:change:context:方法，当keyPath属性发生改变后，KVO会回调这个方法来通知观察者。
3. 当观察者不需要监听时，可以调用removeObserver:forKeyPath:方法将KVO移除。需要注意的是，调用removeObserver需要在观察者消失之前，否则会导致Crash。

例如，我们定义一个 YZPerson 类 继承自 NSObject ，里面有name 和 age 两个属性

~~~~
@interface YZPerson : NSObject
@property (nonatomic ,assign) int age;
@property (nonatomic,strong) NSString  *name;
@end

~~~~
然后在ViewController中，写如下代码

~~~~
- (void)viewDidLoad {
    [super viewDidLoad];
   	//调用方法
    [self setNameKVO];
}

-(void)setNameKVO{
    self.person = [[YZPerson alloc] init];
    // 注册观察者
    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    
    [self.person addObserver:self forKeyPath:@"name" options:options context:@"1111"];
 
}

// 当监听对象的属性值发生改变时，就会调用
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    NSLog(@"监听到%@的%@属性值改变了 - %@ - %@", object, keyPath, change, context);
}

-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
  self.person.name = @"ccc";

}

-(void)dealloc
{
    // 移除监听
    [self.person removeObserver:self forKeyPath:@"name"];
}

~~~~
执行之后结果为

~~~~
KVOdemo[11482:141804] 监听到<YZPerson: 0x6000004e8400>的name属性值改变了 - {
    kind = 1;
    new = ccc;
    old = "<null>";
} - 1111- 1111
~~~~

### 注意点
**需要注意的是，上面代码中我们已经移除了监听，如果再次移除的话，就会crash**

例如

~~~~

- (void)viewDidLoad {
    [super viewDidLoad];
   	//调用方法
    [self setNameKVO];
}
-(void)setNameKVO{
   self.person = [[YZPerson alloc] init];
    // 注册观察者
    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    
    [self.person addObserver:self forKeyPath:@"name" options:options context:@"1111"];
       // 移除监听
    [person removeObserver:self forKeyPath:@"name"];
    // 再次移除
     [person removeObserver:self forKeyPath:@"name"];

}
~~~~
移除多次会报错 

~~~~
KVOdemo[9261:2171323] *** Terminating app due to uncaught exception 'NSRangeException', 
reason: 'Cannot remove an observer <ViewController 0x139e07220> for the key path "name" 
from <YZPerson 0x281322f20> because it is not registered as an observer.'
~~~~


**如果忘记移除的话，有可能下次收到这个属性的变化的时候，会carsh**

所以，我们要保证add和remove是成对出现的



## 抛出疑问

加入我们又两个YZPerson对象，只监听其中一个

~~~~

- (void)viewDidLoad {
    [super viewDidLoad];
    [self setNameKVO];
}

-(void)setNameKVO{
    
    self.person = [[YZPerson alloc] init];
    self.person2 = [[YZPerson alloc] init];
    // 注册观察者
    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    
    [self.person addObserver:self forKeyPath:@"name" options:options context:@"1111"];
  

}
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    self.person.name = @"ccc";
    self.person2.name = @"ddd";

}
// 当监听对象的属性值发生改变时，就会调用
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    NSLog(@"监听到%@的%@属性值改变了 - %@ - %@", object, keyPath, change, context);
}
- (void)dealloc
{
    // 移除监听
    [self.person removeObserver:self forKeyPath:@"name"];
}
~~~~

点击屏幕时候，打印如下

~~~~
监听到<YZPerson: 0x600001afa740>的name属性值改变了 - {
    kind = 1;
    new = ccc;
    old = "<null>";
} - 1111
~~~~

但是我们知道，

~~~~
 self.person.name = @"ccc";
 self.person2.name = @"ddd";
~~~~

上面这两句代码都是调用  setName

~~~~
@implementation YZPerson
- (void)setName:(NSString *)name{
    _name = name;
}
~~~~

也就是说，两个对象，都是调用 setName 方法，根据iOS的机制，应该都是根据 YZPerson的isa指针，去类对象中查找方法。怎么就能做到 self.person.name 可以监听 self.person2.name 不能监听呢？

## 本质分析

针对上面的疑问，我们可以猜测，是不是person 和 person2 的isa指针不一样呢，导致执行的方法不同呢？

打断点，并打印两者的isa

~~~~
(lldb) po self.person->isa
NSKVONotifying_YZPerson

(lldb) po self.person2->isa
YZPerson

~~~~

发现果然是isa指针不同，既然isa指向不同了。是不是说明两者的类对象不同呢？答案是肯定的。因为oc中，就是根据isa去查找类对象的，那么接下来进行验证

## 验证

### 对类对象进行验证

导入 runtime，对两者的类进行打印

~~~~

  NSLog(@"person添加KVO监听之前 - %@ %@",
          object_getClass(self.person),
          object_getClass(self.person2));
    // 注册观察者
    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    [self.person addObserver:self forKeyPath:@"name" options:options context:@"1111"];
    
    NSLog(@"person添加KVO监听之后 - %@ %@",
          object_getClass(self.person),
          object_getClass(self.person2));


~~~~

打印结果为

~~~~
 KVOdemo[13302:171740] person添加KVO监听之前 - YZPerson YZPerson
 KVOdemo[13302:171740] person添加KVO监听之后 - NSKVONotifying_YZPerson YZPerson
~~~~

由此可见，添加KVO监听之后，确实 self.person 的类对象是NSKVONotifying_YZPerson 而self.person2的类对象不变，依然是 YZPerson

### 注意点：如果使用 [self.person class] 无法获取真实的类

例如我们在添加KVO监听之后，这样来获取类对象

~~~~

 NSLog(@"person添加KVO监听之后 - %@ %@",
              [self.person class],
              [self.person2 class]);

~~~~

那么打印结果为

~~~~
KVOdemo[17839:239214] person添加KVO监听之后 - YZPerson YZPerson
~~~~

这是因为，苹果为我们生成了中间类 NSKVONotifying_YZPerson 但是，他并不想让我们知道有这个类的存在，重写了这个 NSKVONotifying_YZPerson 的class方法，所以，我们获取的结果是不准确的。

### 对方法IMP进行验证

我们知道，当改变name属性的时候，是调用setName: 进行的，那我们就来查看一下setName: 有什么变化

~~~~

  NSLog(@"person添加KVO监听之前 - %p %p",
          [self.person methodForSelector:@selector(setName:)],
          [self.person2 methodForSelector:@selector(setName:)]);
    // 注册观察者
    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    [self.person addObserver:self forKeyPath:@"name" options:options context:@"1111"];
    
    NSLog(@"person添加KVO监听之后 - %p %p",
          [self.person methodForSelector:@selector(setName:)],
          [self.person2 methodForSelector:@selector(setName:)]);

~~~~

结果为

~~~~

 KVOdemo[13655:177448] person添加KVO监听之前 - 0x10ccfa630 0x10ccfa630
 KVOdemo[13655:177448] person添加KVO监听之后 - 0x10d056d1a 0x10ccfa630

~~~~

有上面打印结果可知，添加监听之后，self.person的 setName 地址变了。继续通过LLDB查看

~~~~

KVOdemo[13655:177448] person添加KVO监听之前 - 0x10ccfa630 0x10ccfa630
KVOdemo[13655:177448] person添加KVO监听之后 - 0x10d056d1a 0x10ccfa630
(lldb) p (IMP)0x10ccfa630
(IMP) $0 = 0x000000010ccfa630 (KVOdemo`-[YZPerson setName:] at YZPerson.m:12)
(lldb) p (IMP)0x10d056d1a
(IMP) $1 = 0x000000010d056d1a (Foundation`_NSSetObjectValueAndNotify)

~~~~

可知，添加KVO监听之后，setName:方法指向了 Foundation 框架中的 _NSSetObjectValueAndNotify


### 元类对象验证

既然添加KVO监听之后，类对象不是同一个，那元类对象呢？如下验证

~~~~

 // 注册观察者
    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    [self.person addObserver:self forKeyPath:@"name" options:options context:@"1111"];
    
    
NSLog(@"类对象 - %@ %@",
          object_getClass(self.person),  // self.person.isa
          object_getClass(self.person2)); // self.person2.isa
          
 NSLog(@"元类对象 - %@ %@",
          object_getClass(object_getClass(self.person)), // self.person.isa.isa
          object_getClass(object_getClass(self.person2))); // self.person2.isa.isa
          
~~~~

结果为

~~~~

 KVOdemo[13655:177448] 类对象 - NSKVONotifying_YZPerson YZPerson
 KVOdemo[13655:177448] 元类对象 - NSKVONotifying_YZPerson YZPerson

~~~~

可知，元类对象变成了 NSKVONotifying_YZPerson


## 内部调用流程

那设置了kvo监听之后，内部调用有什么流程呢？我们在Person中添加如下代码

~~~~

#import "YZPerson.h"

@implementation YZPerson
- (void)setName:(NSString *)name{
     _name = name;   
}

- (void)willChangeValueForKey:(NSString *)key
{
    [super willChangeValueForKey:key];
    
    NSLog(@"willChangeValueForKey");
}

- (void)didChangeValueForKey:(NSString *)key
{
    NSLog(@"didChangeValueForKey - begin");
    
    [super didChangeValueForKey:key];
    
    NSLog(@"didChangeValueForKey - end");
}

~~~~

点击屏幕之后，如下打印

~~~~
KVOdemo[17486:233248] willChangeValueForKey
KVOdemo[17486:233248] didChangeValueForKey - begin
KVOdemo[17486:233248] 监听到<YZPerson: 0x600000889ca0>的name属性值改变了 - {
    kind = 1;
    new = ccc;
    old = "<null>";
} - 1111
KVOdemo[17486:233248] didChangeValueForKey - end

~~~~

也就是说调用 [super didChangeValueForKey:key]; 的时候，监听到监听对象的改变，进而处理监听逻辑

## 窥探 NSKVONotifying_YZPerson 的方法

~~~~
- (void)printMethodNamesOfClass:(Class)cls
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

-(void)setNameKVO{
    
    self.person = [[YZPerson alloc] init];

    // 注册观察者
    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    [self.person addObserver:self forKeyPath:@"name" options:options context:@"1111"];
    NSLog(@"person添加KVO监听之后 - self.person的类是：%@   里面的方法有：",object_getClass(self.person));
    [self printMethodNamesOfClass:object_getClass(self.person)];
}

~~~~

上面代码执行结果为

~~~~

KVOdemo[19286:259546] person添加KVO监听之后 - self.person的类是：NSKVONotifying_YZPerson   里面的方法有：
KVOdemo[19286:259546] NSKVONotifying_YZPerson setName:, class, dealloc, _isKVOA,
~~~~

这也进一步验证了，系统重写了新建的子类  **NSKVONotifying_YZPerson** 的setName, class, dealloc，新增了 _isKVOA方法


## 手动调用KVO

由上面可知，KVO监听的关键 **willChangeValueForKey** 和 **didChangeValueForKey** 起了关键作用，一般来说只有监听属性发生变化的时候，才能触发监听，但是如果我们想自己手动调用KVO的话，只要自己手动调用这两个方法就可以了。eg:

~~~~
-(void)setNameKVO{
    
    self.person = [[YZPerson alloc] init];
    // 注册观察者
    NSKeyValueObservingOptions options = NSKeyValueObservingOptionNew | NSKeyValueObservingOptionOld;
    [self.person addObserver:self forKeyPath:@"name" options:options context:@"1111"];
    NSLog(@"person添加KVO监听之后 - self.person的类是：%@   里面的方法有：",object_getClass(self.person));
    [self printMethodNamesOfClass:object_getClass(self.person)];
}
-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    //    self.person.name = @"ccc";
    // 手动调用KVO
    [self.person willChangeValueForKey:@"name"];
    
    [self.person didChangeValueForKey:@"name"];
}
// 当监听对象的属性值发生改变时，就会调用
- (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary *)change context:(void *)context
{
    NSLog(@"监听到%@的%@属性值改变了 - %@ - %@", object, keyPath, change, context);
}
- (void)dealloc
{
    // 移除监听
    [self.person removeObserver:self forKeyPath:@"name"];
}

~~~~

每次点击屏幕的时候，打印如下

~~~~

监听到<YZPerson: 0x600003e5b020>的name属性值改变了 - {
    kind = 1;
    new = "<null>";
    old = "<null>";
} - 1111
~~~~

可以看到虽然，new 和 old都是null ，也就是name的值没有改变，但是因为我们手动调用了,

~~~~
 [self.person willChangeValueForKey:@"name"];
    
 [self.person didChangeValueForKey:@"name"];
~~~~
所以就是会触发KVO




## 拓展深入

### iOS用什么方式实现对一个对象的KVO？(KVO的本质是什么？)

- 利用RuntimeAPI动态生成一个子类，并且让instance对象的isa指向这个全新的子类
当修改instance对象的属性时，会调用Foundation的_NSSetXXXValueAndNotify函数
willChangeValueForKey:
父类原来的setter
didChangeValueForKey:
内部会触发监听器（Oberser）的监听方法( observeValueForKeyPath:ofObject:change:context:）


### 如何手动触发KVO？
- 手动调用willChangeValueForKey:和didChangeValueForKey:

### 直接修改成员变量会触发KVO么？
- 不会触发KVO

因为，触发KVO是因为，执行set方法时候，调用 **willChangeValueForKey** **didChangeValueForKey** 但是直接修改成员变量不会调用set方法

eg:
我们把name 成员变量 设置为如下的形式，就不会自动生成set 和 get方法

~~~~
@interface YZPerson : NSObject
{
    @public
    NSString *_name;
    
}
@property (nonatomic ,assign) int age;

@end
~~~~

在监听控制器里面，改成如下操作,直接修改成员变量

~~~~

-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    
    self.person->_name = @"abc";

}

~~~~

这样是不会触发KVO的，如果我们想让它触发KVO，就手动调用，如下

~~~~
@implementation ViewController

-(void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event{
    
    [self.person willChangeValueForKey:@"name"];
    self.person->_name = @"abc";
	 [self.person didChangeValueForKey:@"name"];
}

@end
~~~~

这样就可以触发KVO了。



### 通过KVC修改属性会触发KVO么？

- 会触发KVO
- 详细分析见 [KVC那点儿事](https://ityongzhen.github.io/KVC%E9%82%A3%E7%82%B9%E5%84%BF%E4%BA%8B.html)


### KVC 与 KVO 的不同？
KVC(键值编码)，即 Key-Value Coding，一个非正式的 Protocol，使用字符串(键)访问一个对象实例变量的机制。而不是通过调用 Setter、Getter 方法等显式的存取方式去访问。
KVO(键值监听)，即 Key-Value Observing，它提供一种机制,当指定的对象的属性被修改后,对象就会接受到通知，前提是执行了 setter 方法、或者使用了 KVC 赋值。

### KVO和 notification(通知)的区别？
notification 比 KVO 多了发送通知的一步。
两者都是一对多，但是对象之间直接的交互，notification 明显得多，需要notificationCenter 来做为中间交互。而 KVO 如我们介绍的，设置观察者->处理属性变化，至于中间通知这一环，则隐秘多了，只留一句“交由系统通知”，具体的可参照以上实现过程的剖析。

notification 的优点是监听不局限于属性的变化，还可以对多种多样的状态变化进行监听，监听范围广，例如键盘、前后台等系统通知的使用也更显灵活方便。
（参照通知机制第五节系统通知名称内容）

### KVO与 delegate 的不同？
和 delegate 一样，KVO 和 NSNotification 的作用都是类与类之间的通信。但是与 delegate 不同的是：
这两个都是负责发送接收通知，剩下的事情由系统处理，所以不用返回值；而 delegate 则需要通信的对象通过变量(代理)联系；
delegate 一般是一对一，而这两个可以一对多。


本文相关代码github地址 [github](https://github.com/ITyongzhen/MyBlogs-iOS-Demos)

本文参考资料:


[Runtime源码](https://opensource.apple.com/tarballs/objc4/)

[iOS底层原理](https://ke.qq.com/course/package/11609)

[KVO原理分析及使用进阶](https://segmentfault.com/a/1190000013813643)

[FackBook的KVOController](https://github.com/facebook/KVOController) 

[iOS开发 -- KVO的实现原理与具体应用](https://www.jianshu.com/p/e59bb8f59302)




