---
layout: post
title: Swift之协议
date: 2019-10-16 17:32:24.000000000 +09:00
categories: 
- Swift
---



本文首发于[个人博客](https://ityongzhen.github.io/Swift%E4%B9%8B%E5%8D%8F%E8%AE%AE.html)

## 前言

协议，有关开发经验的应该都不陌生，很多语言中都有协议，但是相对来说，Swift中的协议更加强大，灵活。

- Swift中协议可以用来定义方法、属性、下标的声明，协议可以被枚举、结构体、类遵守（多个协议之间用逗号隔开）

~~~~
//协议
protocol Drawable {
    //方法
    func draw()
    //可读可写属性
    var x: Int { get set }
    //只读属性
    var y: Int { get }
    // 下标
    subscript(index: Int) -> Int { get set }
}

~~~~

类`TestClass `准守多个协议

~~~~
protocol Test1 { }
protocol Test2 { }
protocol Test3 { }
class TestClass : Test1, Test2, Test3 { }
~~~~


### 需要注意的是

- 协议中定义方法时不能有默认参数值
- 默认情况下，协议中定义的内容必须全部都实现
- 也有办法办到只实现部分内容，后面会说

## 协议中的属性

- 协议中定义属性时必须用var关键字
- 实现协议时的属性权限要不小于协议中定义的属性权限
- 协议定义get、set，用var存储属性或get、set计算属性去实现
- 协议定义get，用任何属性都可以实现

eg：有协议`Drawable`,里面有方法`draw `，以及可读可写属性`x`，只读属性`y`，下标。其中属性必须用`var`关键字

~~~~
//协议
protocol Drawable {
    func draw() //方法
    var x: Int { get set } //可读可写 属性用var
    var y: Int { get } //只读  属性用var
    subscript(index: Int) -> Int { get set } //下标
}
~~~~

当实现的时候，有如下的方式，

~~~~
class Person : Drawable {
    var x: Int = 0 //用var的存储属性
    let y: Int = 0 //let实现只读属性
    func draw() {
        print("Person draw")
    }
    subscript(index: Int) -> Int {
        set { }
        get { index }
    }
}
~~~~


当然了。也可以写成如下这种

~~~~
class Person : Drawable {
    var x: Int { //用计算属性
        get { 0 }
        set { }
    }
    var y: Int { 0 } //var实现只读属性
    func draw() { print("Person draw") }
    subscript(index: Int) -> Int {
        set { }
        get { index }
    }
}
~~~~

## static、class

- 为了保证通用，协议中必须用static定义类型方法、类型属性、类型下标
	- 因为class只能用在类中，不能用于结构体等。所以为了通用，用static
	- 但是实现的时候，可以用class，也可以用static，具体看自己的情况
eg:

~~~~
protocol Drawable {
	//这里必须用static
    static func draw()
}
class Person1 : Drawable {
	//这里可以用class
    class func draw() {
        print("Person1 draw")
    }
}
class Person2 : Drawable {
	//这里也可以用static
    static func draw() {
        print("Person2 draw")
    }
}
~~~~


## mutating

关于`mutating`可以参考[Swift之方法](https://juejin.im/post/5d2ba7265188252d1d5f9218)

- 只有将协议中的实例方法标记为mutating
	- 才允许结构体、枚举的具体实现修改自身内存
	- 类在实现方法时不用加mutating，枚举、结构体才需要加mutating

eg:

~~~~
protocol Drawable {
    mutating func draw()
}

class Size : Drawable {
    var width: Int = 0
    func draw() {
        width = 10
    }
}

struct Point : Drawable {
    var x: Int = 0
    mutating func draw() {
        x = 10
    }
}
~~~~

## init

- 协议中还可以定义初始化器init
	- 非final类实现时必须加上required

可以这么理解，如果定义的类，有子类，那么子类必须准守初始化器init，所以加上关键字`required `,但是，如果一个被`final `修饰的类。就不用加上`required `.因为被`final `修饰的类不能被其他类继承。

关于`final`可参考[Swift之继承](https://juejin.im/post/5d2d9ebbe51d4510634318bf)


eg: 有协议`Drawable `,里面定义了初始化器`init`，类Point遵守这个协议，所以在init 前面加了关键字 `required `,这样，继承类Point的子类都要实现这个方法，但是类Size没子类。因为加了关键字`final `，这个类不能被继承。所以`init `前面不用加`required `

~~~~
protocol Drawable {
    init(x: Int, y: Int)
}
class Point : Drawable {
    required init(x: Int, y: Int) { }
}
final class Size : Drawable {
    init(x: Int, y: Int) { }
}
~~~~

- 如果从协议实现的初始化器，刚好是重写了父类的指定初始化器
	- 那么这个初始化必须同时加required、override

eg:

~~~~
protocol Livable {
    init(age: Int)
}

class Person {
    init(age: Int) { }
}

class Student : Person, Livable {
    required override init(age: Int) {
        super.init(age: age)
    }
}
~~~~


## init、init?、init!
- 协议中定义的init?、init!，可以用init、init?、init!去实现
- 协议中定义的init，可以用init、init!去实现

eg:

~~~~
//协议
protocol Livable {
    init()
    init?(age: Int)
    init!(no: Int)
}

//类
class Person : Livable {
	
	//下面两种都可以实现init()
    required init() { }
    // required init!() { }
    
    //下面3种都可以实现init?(age: Int)
    required init?(age: Int) { }
    // required init!(age: Int) { }
    // required init(age: Int) { }
    
     //下面3种都可以实现 init!(no: Int)
    required init!(no: Int) { }
    // required init?(no: Int) { }
    // required init(no: Int) { }
}

~~~~


## 协议的继承

- 一个协议可以继承其他协议

eg

~~~~
// 协议Runnable
protocol Runnable {
    func run()
}

// 协议Livable 继承协议 Runnable
protocol Livable : Runnable {
    func breath()
}

class Person : Livable {
    func breath() { }
    func run() { }
}

~~~~

## 协议组合

- 协议组合，多个协议组合在一起，而且可以包含1个类类型（最多1个）

eg: 两个协议`Livable `和`Runnable `,类`Person `

~~~~
protocol Livable { } n
protocol Runnable { }
class Person { }
~~~~

下面定义了fn0,接收参数必须是Person或者其子类的实例。fn1接收参数必须遵守Livable协议的实例，其他的可以自行看代码

~~~~
// 接收Person或者其子类的实例
func fn0(obj: Person) { }


// 接收遵守Livable协议的实例
func fn1(obj: Livable) { }


// 接收同时遵守Livable、Runnable协议的实例
func fn2(obj: Livable & Runnable) { }


// 接收同时遵守Livable、Runnable协议、并且是Person或者其子类的实例
func fn3(obj: Person & Livable & Runnable) { }


typealias RealPerson = Person & Livable & Runnable
// 接收同时遵守Livable、Runnable协议、并且是Person或者其子类的实例
func fn4(obj: RealPerson) { }
~~~~

## CaseIterable

- 让枚举遵守CaseIterable协议，可以实现遍历枚举值

~~~~
// 枚举Season遵守协议CaseIterable
enum Season : CaseIterable {
    case spring, summer, autumn, winter
}

// 取出所有的case
let seasons = Season.allCases
print(seasons.count) // 4

// 可以遍历
for season in seasons {
    print(season)
} // spring summer autumn winter
~~~~

## CustomStringConvertible

- 遵守CustomStringConvertible协议，可以自定义实例的打印字符串

eg: ` Person `类遵守了`CustomStringConvertible`协议,可以再内部自定义打印`description `。

~~~~
class Person : CustomStringConvertible {
    var age: Int
    var name: String
    init(age: Int, name: String) {
        self.age = age
        self.name = name
    }
    var description: String {
        "age=\(age), name=\(name)"
    }
}
var p = Person(age: 10, name: "Jack")
print(p) // age=10, name=Jack

~~~~

## Any、AnyObject

- Swift提供了2种特殊的类型：Any、AnyObject
	- Any：可以代表任意类型（枚举、结构体、类，也包括函数类型）
	- AnyObject：可以代表任意类类型（在协议后面写上: AnyObject代表只有类能遵守这个协议）

eg: stu属于Any类型,可以赋值为字符串或者对象等

~~~~
var stu: Any = 10
stu = "Jack"
stu = Student()
~~~~

eg: 创建1个能存放任意类型的数组

~~~~
// 创建1个能存放任意类型的数组
// 第一种写法
// var data = Array<Any>()
// 第二种写法
var data = [Any]()
data.append(1)
data.append(3.14)
data.append(Student())
data.append("Jack")
data.append({ 10 })
~~~~


## is、as?、as!、as

- is用来判断是否为某种类型，as用来做强制类型转换

eg: 定义协议`Runnable `,类`Person`和类`Student`

~~~~
protocol Runnable { func run() }
class Person { }
class Student : Person, Runnable {
    func run() {
        print("Student run")
    }
    func study() {
        print("Student study")
    }
}
~~~~

使用is 的时候

~~~~
var stu: Any = 10
print(stu is Int) // true
stu = "Jack"
print(stu is String) // true
stu = Student()
print(stu is Person) // true
print(stu is Student) // true
print(stu is Runnable) // t
~~~~

使用as 的时候,如果转换失败，后面都不执行。转换成功，后面才继续执行

~~~~
var stu: Any = 10
(stu as? Student)?.study() // 没有调用study
stu = Student()
(stu as? Student)?.study() // Student study
(stu as! Student).study() // Student study
(stu as? Runnable)?.run() // Student run
~~~~


## X.self、X.Type、AnyClass

- X.self是一个元类型（metadata）的指针，metadata存放着类型相关信息
- X.self属于X.Type类型

~~~~
// 定义类Person
class Person { }
// 定义类Student 继承 Person
class Student : Person { }
//perType类型是 Person.Type
var perType: Person.Type = Person.self
//stuType类型是 Student.Type
var stuType: Student.Type = Student.self
// Student.self可以赋值给perType
perType = Student.self


// anyType可以是任何类型
var anyType: AnyObject.Type = Person.self
anyType = Student.self


public typealias AnyClass = AnyObject.Type
//anyType2可以是任何类型
var anyType2: AnyClass = Person.self
anyType2 = Student.self
~~~~

## 元类型的应用

eg:`Cat`类，`Dog `类，`Pig `类，都继承自`Animal `类，我们想同时去初始化，可使用下面的代码

~~~~
class Animal { required init() { } }
class Cat : Animal { }
class Dog : Animal { }
class Pig : Animal { }
func create(_ clses: [Animal.Type]) -> [Animal] {
    var arr = [Animal]()
    for cls in clses {
        // 根据元类型初始化
        arr.append(cls.init())
    }
    return arr
}
// 这里传入Cat,Dog,Pig进行初始化
print(create([Cat.self, Dog.self, Pig.self]))
~~~~

**注意上面的required不能省略**

- 在OC、Java等语言中，任何一个类最终都要继承自某个基类，
- 在Swift中没有这个规定，如果一个类不继承任何一个类，那这个类就是基类

实际上，真的如此么，真的不继承任何类么？

~~~~
class Person {
    var age: Int = 0
}
class Student : Person {
    var no: Int = 0
}
print(class_getInstanceSize(Student.self)) // 32
print(class_getSuperclass(Student.self)!) // Person
print(class_getSuperclass(Person.self)!) // Swift._SwiftObject
~~~~


- 从结果可以看得出来，Swift还有个隐藏的基类：Swift._SwiftObject
- 可以参考[Swift源码](https://github.com/apple/swift/blob/master/stdlib/public/runtime/SwiftObjec.h)


## Self
- Self一般用作返回值类型，限定返回值跟方法调用者必须是同一类型（也可以作为参数类型）

有点类似OC中的instanType的感觉。eg:

~~~~
protocol Runnable {
    func test() -> Self
}
class Person : Runnable {
    required init() { }
    func test() -> Self { type(of: self).init() }
}
class Student : Person { }
~~~~

调用时候

~~~~
var p = Person()
// 输出Person
print(p.test())


var stu = Student()
// 输出Student
print(stu.test())
~~~~


参考资料：



[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

