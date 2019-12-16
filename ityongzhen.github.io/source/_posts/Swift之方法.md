---
layout: post
title: Swift之方法
date: 2019-08-30 17:32:24.000000000 +09:00
categories: 
- Swift
---

本文首发于[个人博客](https://ityongzhen.github.io/Swift%E4%B9%8B%E6%96%B9%E6%B3%95.html)

## 前言

方法，也就是函数。同其他语言一样，在Swift中，也是分为实例方法和类型方法


## 枚举、结构体、类都可以定义实例方法、类型方法

### 实例方法（Instance Method）：通过实例对象调用

### 类型方法（Type Method）：通过类型调用，用static或者class关键字定义，类似OC中的类方法

例如有个类Car,有实例方法`getnNum `和类型方法`getCount`

~~~~
class Car {
    static var cout = 0
    var num = 0
    init() {
        Car.cout += 1
    }
    // 类型方法
    static func getCount() -> Int { cout }
    // 实例方法
    func getnNum() -> Int {
        num
    }
}
let c0 = Car()
let c1 = Car()
let c2 = Car()
print(Car.getCount()) // 3

c0.num = 10
c1.num = 11

print(c1.num) //11
print(c2.num) //0
~~~~

### self
不管是实例方法，还是类型方法，里面都可以调用 `self`

- 在实例方法中代表实例对象
- 在类型方法中代表类型
	- 在类型方法static func getCount中
	- cout等价于self.cout、Car.self.cout、Car.cout

例如上面的代码可以写成

~~~~
class Car {
    static var cout = 0
    var num = 0
    init() {
        Car.cout += 1
    }
    // 类型方法
    static func getCount() -> Int {
        self.cout //self代表类型
    }
    
    func getnNum() -> Int {
        self.num //self代表实例
    }
}
~~~~

## 关键字`mutating`
- 结构体和枚举是值类型，默认情况下，值类型的属性不能被自身的实例方法修改
- 在func关键字前加mutating可以允许这种修改行为

eg:

~~~~
struct Point {
    var x = 0.0, y = 0.0
     func moveBy(deltaX: Double, deltaY: Double) {
        x += deltaX //编译报错 Left side of mutating operator isn't mutable: 'self' is immutable
        y += deltaY //编译报错 Left side of mutating operator isn't mutable: 'self' is immutable
         self = Point(x: x + deltaX, y: y + deltaY) //编译报错 Cannot assign to value: 'self' is immutable
    }
}



enum StateSwitch {
    case low, middle, high
     func next() {
        switch self {
        case .low:
            self = .middle//编译报错 Cannot assign to value: 'self' is immutable
        case .middle:
            self = .high//编译报错 Cannot assign to value: 'self' is immutable
        case .high:
            self = .low//编译报错 Cannot assign to value: 'self' is immutable
        }
    }
}
~~~~

- 加上关键字`mutating`之后就可以了

~~~~
struct Point {
    var x = 0.0, y = 0.0
    mutating func moveBy(deltaX: Double, deltaY: Double) {
        x += deltaX 
        y += deltaY 
        self = Point(x: x + deltaX, y: y + deltaY) 
      }
}



enum StateSwitch {
    case low, middle, high
   mutating  func next() {
        switch self {
        case .low:
            self = .middle
        case .middle:
            self = .high
        case .high:
            self = .low        }
    }
}
~~~~

## 关键字`@discardableResult`


- 在func前面加个@discardableResult，可以消除：函数调用后返回值未被使用的警告⚠

eg:

~~~~

struct Point {
    var x = 0.0, y = 0.0
     mutating func moveX(deltaX: Double) -> Double {
        x += deltaX
        return x
    }
}
var p = Point()
p.moveX(deltaX: 10)
~~~~

因为方法`moveX `的返回值没有使用，编译器会报警告

![](https://user-gold-cdn.xitu.io/2019/7/15/16bf284e958adef2?w=871&h=216&f=png&s=50207)


- 如果加了关键字`@discardableResult`就不会警告了

~~~~

struct Point {
    var x = 0.0, y = 0.0
    @discardableResult mutating 
    func moveX(deltaX: Double) -> Double {
        x += deltaX
        return x
    }
}
var p = Point()
p.moveX(deltaX: 10)
~~~~




参考资料:


[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

