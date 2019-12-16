---
layout: post
title: Swift之属性
date: 2019-08-20 17:32:24.000000000 +09:00
categories: 
- Swift
---



本文首发于[我的个人博客](https://ityongzhen.github.io/Swift之属性.html)

## 前言

前面了解了 [Swift之类](https://juejin.im/post/5d216d6bf265da1b957079d8)  和 [Swift之结构体](https://juejin.im/post/5d216e1ff265da1bb27752aa)  

这篇文章分享一下Swift中的属性



## Swift中跟实例相关的属性可以分为2大类

### 存储属性（Stored Property）

- 类似于成员变量这个概念
- 存储在实例的内存中
- 结构体、类可以定义存储属性
- 枚举不可以定义存储属性
	- 关于枚举不可以定义存储属性，根据之前的[Swift枚举](https://juejin.im/post/5d1eeca3f265da1ba6480cc7)一文可知，
枚举中存储关联值或者keys，不存储属性的。

#### 关于存储属性，Swift有个明确的规定

- 在创建类 或 结构体的实例时，必须为所有的存储属性设置一个合适的初始值
	- 可以在初始化器里为存储属性设置一个初始值
	- 可以分配一个默认的属性值作为属性定义的一部分

关于这个规定，我们在[Swift之结构体](https://juejin.im/post/5d216e1ff265da1bb27752aa)   一文中已经说过了，这里稍微提一下，比如下面代码，x和y都是存储属性，当初始化的时候，如果没值，编译器会直接报错。

~~~~
struct Point{
    var x: Int
    var y: Int
}

var p1 = Point(x: 10, y: 20)
var p2 = Point(y: 20) //报错 Missing argument for parameter 'x' in call
var p3 = Point(x: 10) //报错 Missing argument for parameter 'y' in call
var p4 = Point()    //报错 Missing argument for parameter 'x' in call
~~~~

### 计算属性（Computed Property）

- 本质就是方法（函数）
- 不占用实例的内存
- 枚举、结构体、类都可以定义计算属性
- set传入的新值默认叫做newValue，也可以自定义


如下面结构体`Circle`包括了存储属性`radius `和计算属性`diameter `

~~~~
truct Circle {
    // 存储属性
    var radius: Double
    // 计算属性
    var diameter: Double {
        set {
            radius = newValue / 2
        }
        get {
           return radius * 2
        }
    }
}

var circle = Circle(radius: 5)
print(circle.radius) // 5.0
print(circle.diameter) // 10.0
circle.diameter = 12
print(circle.radius) // 6.0
print(circle.diameter) // 12.0
~~~~

- 只读计算属性：只有get，没有set
如下

~~~~
struct Circle {
    var radius: Double
    var diameter: Double {
        get {
            radius * 2
        }
    }
}
~~~~

只读计算属性可以简写，例如上面的代码可以如下表示

~~~~
struct Circle {
    var radius: Double
    var diameter: Double { radius * 2 }
}
~~~~


#### 计算属性不占用实例的内存

关于不占用实例的内存，可以如下代码证明

~~~~
struct Circle {
    // 存储属性
    var radius: Double
    // 计算属性
    var diameter: Double {
        set {
            radius = newValue / 2
        }
        get {
           return radius * 2
        }
    }
}
print("Double占用字节", MemoryLayout<Double>.stride)
print("Circle占用字节",MemoryLayout<Circle>.stride) // 8

~~~~

输出为

~~~~
Double占用字节 8
Circle占用字节 8
~~~~

也就是说`Circle`占用的仅仅是其存储属性`radius `所占用的内存。和计算属性无关的，读者也可以多写几个计算属性，自行验证。




### 汇编分析存储属性和计算属性的区别

代码如下

~~~~
struct Circle {
    // 存储属性
    var radius: Int
    // 计算属性
    var diameter: Int {
        set {
            radius = newValue / 2
        }
        get {
           return radius * 2
        }
    }
}

var circle = Circle(radius: 9)
circle.radius = 5 
circle.diameter = 8 //这里打断点
~~~~

![](https://user-gold-cdn.xitu.io/2019/7/14/16bee94ad12510f0?w=1114&h=800&f=png&s=400156)

- 5存储到了全局变量`0x3e96(%rip)`,全局变量只有`circle`，所以也就是说存储属性的值，被直接放在了结构体的内存中。

- 8赋值给寄存器`%r8d`，又给寄存器%edi，作为参数调用函数`0x100001ae0`的时候传入，而函数`0x100001ae0`就是`testSwift.Circle.diameter.setter `这就是存储属性和计算属性的区别

## 枚举rawValue的原理

在[Swift枚举](https://juejin.im/post/5d1eeca3f265da1ba6480cc7)一文中，我们说过枚举原始值是不占用内存的。

~~~~

enum Season: Int{
    case test1 = 1, test2, test3, test4

}

var s = Season.test2
print(s.rawValue) //输出2

~~~~

上面代码输出为2。
这些原始值 test1 = 1, test2, test3, test4，系统内部完全可以写成只读计算属性

~~~~
enum Season: Int{
    case test1 = 1, test2, test3, test4
    
    var rawValue : Int {
        switch self {
        case .test1:
            return 11
        case .test2:
            return 12
        case .test3:
            return 13
        case .test4:
            return 14

        }
    }
}

var s = Season.test2
print(s.rawValue) //输出12
~~~~

上面代码输出为12，这样就完成了获取枚举的原始值的时候，直接获取的是只读计算属性。光看结果是不够令人信服的，那就看汇编

~~~~

enum Season: Int{
    case test1 = 1, test2, test3, test4

}

var s = Season.test2
print(s.rawValue) //这里打断点

~~~~

上述代码在最后一行打断点

![](https://user-gold-cdn.xitu.io/2019/7/14/16bee94ad1350252?w=1077&h=611&f=png&s=261495)

可以看到，确实调用了`rawValue.getter`。

所以汇编才是看出本质的神器。


## 延迟存储属性（Lazy Stored Property）


### 使用lazy可以定义一个延迟存储属性，在第一次用到属性的时候才会进行初始化
- lazy属性必须是var，不能是let
	- let必须在实例的初始化方法完成之前就拥有值
- 如果多条线程同时第一次访问lazy属性
 	- 无法保证属性只被初始化1次


eg:有类 Car和类Person ，car作为Person的延迟存储属性，那么当使用car的是，才会调用car的初始化方法。

~~~~
class Car {
    init() {
        print("Car init!")
    }
    func run() {
        print("Car is running!")
    }
}
class Person {
    lazy var car = Car()
    init() {
        print("Person init!")
    }
    func goOut() {
        car.run()
    }
}
let p = Person()
print("--------")
p.goOut()

~~~~

输出为

~~~~
Person init!
--------
Car init!
Car is running!
~~~~

### **注意点**
- 当结构体包含一个延迟存储属性时，只有var才能访问延迟存储属性
	- 因为延迟属性初始化时需要改变结构体的内存

## 属性观察器

- 可以为非lazy的var存储属性设置属性观察器

~~~~
struct Circle {
    var radius: Double {
        willSet {
            print("willSet", newValue)
        }
        didSet {
            print("didSet", oldValue, radius)
        }
    }
    init() {
        self.radius = 1.0
        print("Circle init!")
    }
}

var circle = Circle()// 输出 Circle init!

circle.radius = 10.5 
// 输出 willSet 10.5 
// didSet 1.0 10.5

print(circle.radius) //输出 10.5
~~~~

### 注意点

- willSet会传递新值，默认叫newValue
- didSet会传递旧值，默认叫oldValue
- 在初始化器中设置属性值不会触发willSet和didSet
- 在属性定义时设置初始值也不会触发willSet和didSet





## 全局变量、局部变量
- 属性观察器、计算属性的功能，同样可以应用在全局变量、局部变量身上

eg:

~~~~
var num: Int {
    get {
        return 10
    }
    set {
        print("setNum", newValue)
    }
}

num = 11 // setNum 11
print(num) // 10



func test() {
    var age = 10 {
        willSet {
            print("willSet", newValue)
        }
        didSet {
            print("didSet", oldValue, age)
        }
    }
    age = 11
    // willSet 11
    // didSet 10 11
}

test()
~~~~

## 类型属性（Type Property）

### 类型属性分类
 
1. 实例属性（Instance Property）：只能通过实例去访问
	- 存储实例属性（Stored Instance Property）：存储在实例的内存中，每个实例都有1份
	- 计算实例属性（Computed Instance Property）
2. 类型属性（Type Property）：只能通过类型去访问
	- 存储类型属性（Stored Type Property）：整个程序运行过程中，就只有1份内存（类似于全局变量）
	- 计算类型属性（Computed Type Property）
3. 可以通过static定义类型属性
	- 如果是类，也可以用关键字class

eg:

~~~~
struct Car {
    static var count: Int = 0
    init() {
        Car.count += 1
    }
}
let c1 = Car()
let c2 = Car()
let c3 = Car()
print(Car.count) // 输出3
~~~~


### 类型属性细节
1. 不同于存储实例属性，你必须给存储类型属性设定初始值
	- 因为类型没有像实例那样的init初始化器来初始化存储属性
2. 存储类型属性默认就是lazy，会在第一次使用的时候才初始化
	- 就算被多个线程同时访问，保证只会初始化一次
	- 存储类型属性可以是let
3. 枚举类型也可以定义类型属性（存储类型属性、计算类型属性）

## 单例模式

关于单例模式，可以参考我的另一篇文章[你真的懂单例模式么](https://juejin.im/post/5d295106e51d45105d63a5b2)

不同语言的单例模式，都是类似的，这里给出Swift版本单例的实现。

~~~~
import Foundation

public class FileManager {
	//单例模式
    public static let shared = FileManager()
    private init() { }
}

// 如果单例里面代码过多，可以写成如下
public class FileManager {
    public static let shared = {
        // ....
        // ....
        return FileManager()
    }()
    private init() { }
}
~~~~

## 汇编分析 static

下面的代码

~~~~
import Foundation

public class YZPerson {
    static  var count = 3 //这里打断点
}

YZPerson.count = 6 

~~~~

如下图所示，可以看出，会调用 `swift_once`函数，来到这个调用位置，si汇编调试指令，一直跟进去

![](https://user-gold-cdn.xitu.io/2019/7/14/16bee94ad15fe540?w=1950&h=678&f=png&s=272688)


最终会来到这里，调用`dispatch_once_f`
![](https://user-gold-cdn.xitu.io/2019/7/14/16bee94ad1e7961e?w=1932&h=790&f=png&s=288329)

也就是说`static `内部封装了`dispatch_once_` 而`dispatch_once_`能保证线程安全的，只能被初始化一次，所以单例的时候可以用`static ` 关于 `dispatch_once`的分析，可以看这篇文章[你真的懂单例模式么](https://juejin.im/post/5d295106e51d45105d63a5b2)

- 同样的，我们可以从代码角度，汇编角度分别证明 `static` 修饰的变量，属于全局变量。读者有兴趣自己证明。这里不再赘述。


参考资料:


[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

[你真的懂单例模式么](https://juejin.im/post/5d295106e51d45105d63a5b2)






