---
layout: post
title: Swift之继承
date: 2019-09-18 17:32:24.000000000 +09:00
categories: 
- Swift
---


本文首发于[个人博客](https://ityongzhen.github.io/Swift%E4%B9%8B%E7%BB%A7%E6%89%BF.html)

## 前言

同其他语言一样，Swift中也是有继承的



- 值类型（枚举、结构体）不支持继承，只有类支持继承
- 没有父类的类，称为：基类
	- Swift并没有像OC、Java那样的规定：任何类最终都要继承自某个基类
- 子类可以重写父类的下标、方法、属性，重写必须加上`override`关键字


## 类继承的内存结构


 - 有如下`Animal类`,其中`Dog 类`继承`Animal类` ，其中`ErHa 类`继承`Dog类`



~~~~
class Animal {
    var age = 0
}
class Dog : Animal {
    var weight = 0
}
class ErHa : Dog {
    var iq = 0
}
~~~~

### `Animal类`的内存

如下代码查看类的内存大小和相应内存的值，需要用到工具[Mems](https://github.com/CoderMJLee/Mems)
关于的使用可以看[Swift之枚举](https://juejin.im/post/5d1eeca3f265da1ba6480cc7)一文，或者直接看github上[Mems](https://github.com/CoderMJLee/Mems)的说明


~~~~
let a = Animal()
a.age = 5
print(Mems.size(ofRef: a))
print(Mems.memStr(ofRef: a))
~~~~

`Animal类`的内存大小和内容，分别为

~~~~
32
0x00000001000084d8 //存放类的相关信息
0x0000000000000002 //引用技术
0x0000000000000005  // age数值5
0x0000000000000000	//没用到
~~~~

### `Animal类`的内存分析

- 首先类本身就要占用16个字节，其中8个存放类的信息，8个存放引用技术，具体分析见[Swift之类](https://juejin.im/post/5d216d6bf265da1b957079d8)
- 然后是 age的值5 占用8个字节，总共24个字节，
- 因为内存对齐的原因，必须是16的倍数，所以需要32个字节，其中最后8个字节用不到，全是0



### `Dog类`的内存

如下代码查看类的内存大小和相应内存的值


~~~~
let d = Dog()
d.age = 6
d.weight = 7
print(Mems.size(ofRef: d))
print(Mems.memStr(ofRef: d))
~~~~

`Dog类`的内存大小和内容，分别为

~~~~
32
0x0000000100008588 //存放类的相关信息
0x0000000000000002 //引用技术
0x0000000000000006  // age数值6
0x0000000000000007	//weight的值7
~~~~

### `Dog类`的内存分析

- 首先类本身就要占用16个字节，其中8个存放类的信息，8个存放引用技术，具体分析见[Swift之类](https://juejin.im/post/5d216d6bf265da1b957079d8)
- 然后是 age的值6 占用8个字节，weight的值7 占用8个字节， 总共32个字节，
- 因为内存对齐的原因，必须是16的倍数，但是32正好是16的倍数，所以总共需要32个字



### `ErHa类`的内存

如下代码查看类的内存大小和相应内存的值


~~~~
let e = ErHa()
e.age = 8
e.weight = 9
e.iq = 10
print(Mems.size(ofRef: e))
print(Mems.memStr(ofRef: e))
~~~~

`ErHa类`的内存大小和内容，分别为

~~~~
48
0x0000000100008658 //存放类的相关信息
0x0000000000000002 //引用技术
0x0000000000000008  // age数值8
0x0000000000000009	//weight的值9
0x000000000000000a	//iq的值10
0x0000000000000000	//内存对齐增加的，没用到
~~~~

### `ErHa类`的内存分析

- 首先类本身就要占用16个字节，其中8个存放类的信息，8个存放引用技术，具体分析见[Swift之类](https://juejin.im/post/5d216d6bf265da1b957079d8)
- 然后是 age,weight,iq分别占用8个字节，16+24 = 40个字节了
- 因为内存对齐的原因，必须是16的倍数，所以需要48个字节

## 重写实例方法、下标

子类可以重写父类的实例方法、下标

有个`Animal` 类

~~~~
class Animal {
    func speak() {
        print("Animal speak")
    }
    subscript(index: Int) -> Int {
        return index
    }
}
~~~~

子类`Cat`继承`Animal`,如果重写示例方法，下标，必须用关键字`override`

~~~~
class Animal {
    func speak() {
        print("Animal speak")
    }
    subscript(index: Int) -> Int {
        return index
    }
}
class Cat : Animal {
    override func speak() {
        super.speak()
        print("Cat speak")
    }
    override subscript(index: Int) -> Int {
        return super[index] + 1
    }
}
~~~~

## 重写类型方法、下标

上面的实例方法、下标。如果是类型方法、下标的话，有些许不同，

- 被class修饰的类型方法、下标，允许被子类重写
- 被static修饰的类型方法、下标，不允许被子类重写

eg


~~~~
class Animal {
	//static修饰
    static func speak() {
        print("Animal speak")
    }
    // class修饰
    class subscript(index: Int) -> Int {
        return index
    }
}

print(Animal[6])
class Cat : Animal {
	//编译报错 Cannot override static method
    override class func speak() {
        super.speak()
        print("Cat speak")
    }
    //编译成功
    override class subscript(index: Int) -> Int {
        return super[index] + 1
    }
}
~~~~

如上面的代码所示`static `修饰的时候，子类重写，直接报错` Cannot override static method`。而`class `修饰时候，编译正常

## 重写属性

- 子类可以将父类的属性（存储、计算）重写为计算属性
- 子类不可以将父类属性重写为存储属性
- 只能重写var属性，不能重写let属性
- 重写时，属性名、类型要一致
- 子类重写后的属性权限 不能小于 父类属性的权限
	- 如果父类属性是只读的，那么子类重写后的属性可以是只读的、也可以是可读写的
	- 如果父类属性是可读写的，那么子类重写后的属性也必须是可读写的
	
### 重写类型属性
 重写类型属性，比较简单，不做赘述

### 重写类型属性

**注意的是：如果子类把父类的存储属性int 重写为计算属性，子类中依然有8个字节存储该int属性**

**再次需要注意的是，和重写类型方法、下标类似 如果重写类型属性**

- 被class修饰的类型方法、下标，允许被子类重写
- 被static修饰的类型方法、下标，不允许被子类重写

## 属性观察器

### 可以在子类中为父类属性（除了只读计算属性、let属性）增加属性观察器，依然是存储属性

eg，子类`SubCircle `给父类`Circle `的存储属性`radius `增加属性观察器

~~~~
class Circle {
    var radius: Int = 1
}
class SubCircle : Circle {
    override var radius: Int {
        willSet {
            print("SubCircle willSetRadius", newValue)
        }
        didSet {
            print("SubCircle didSetRadius", oldValue, radius)
        }
    }
}

~~~~

**注意点：子类增加属性观察器之后，依然是存储属性**

### 如果父类本身就有属性观察器

eg:

~~~~
class Circle {
    var radius: Int = 1 {
        willSet {
            print("Circle willSetRadius", newValue)
        }
        didSet {
            print("Circle didSetRadius", oldValue, radius)
        }
    }
}
class SubCircle : Circle {
    override var radius: Int {
        willSet {
            print("SubCircle willSetRadius", newValue)
        }
        didSet {
            print("SubCircle didSetRadius", oldValue, radius)
        }
    }
}
var circle = SubCircle()

~~~~

输出为

~~~~
SubCircle willSetRadius 10
Circle willSetRadius 10
Circle didSetRadius 1 10
SubCircle didSetRadius 1 10
~~~~

### 重写父类的计算属性

如下代码

~~~~
class Circle {
    var radius: Int {
        set {
            print("Circle setRadius", newValue)
        }
        get {
            print("Circle getRadius")
            return 20
        }
    }
}
class SubCircle : Circle {
    override var radius: Int {
        willSet {
            print("SubCircle willSetRadius", newValue)
        }
        didSet {
            print("SubCircle didSetRadius", oldValue, radius)
        }
    }
}

~~~~


使用

~~~~
var circle = SubCircle()
circle.radius = 10
~~~~

输出结果为

~~~~
Circle getRadius
SubCircle willSetRadius 10
Circle setRadius 10
Circle getRadius
SubCircle didSetRadius 20 20
~~~~

输出结果分析

- 调用`circle.radius = 10`的时候，先获取了`oldValue `，调用父类的`get `输出`Circle getRadius`
- 然后调用子类`willSetRadius `准备赋值
- 调用父类`setRadius `
- 准备调用子类的` print("SubCircle didSetRadius", oldValue, radius)`之前，要先获取`radius `的值，所以，需要先执行父类的`getRadius `
- 执行子类的`didSet`方法


## final关键字

- 被final修饰的方法、下标、属性，禁止被重写
- 被final修饰的类，禁止被继承



参考资料:


[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

