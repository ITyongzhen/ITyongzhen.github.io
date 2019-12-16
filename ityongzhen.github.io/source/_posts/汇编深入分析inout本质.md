---
layout: post
title: 汇编深入分析inout本质
date: 2019-09-28 17:32:24.000000000 +09:00
categories: 
- Swift
---



本文首发于[个人博客](https://ityongzhen.github.io/%E6%B1%87%E7%BC%96%E6%B7%B1%E5%85%A5%E5%88%86%E6%9E%90inout%E6%9C%AC%E8%B4%A8.html)

## 前言

关于`输入输出参数inout` 在[Swift之函数](https://juejin.im/post/5d1eec54f265da1bb27750ba)一文中，我们已经有了初步的认识。现在我们再继续深入了解一下

## 输入输出参数

- 说了形参只能是let，但是如果我们想再内部修改外部实参的值，可以用 inout 定义输入输出参数

例如

~~~~
   func swapValues(_ v1: inout Int, _ v2: inout Int) {
        let tmp = v1
        v1 = v2
        v2 = tmp
        //前面三行也可以换成  (v1, v2) = (v2, v1) 效果一样
    }
    
     var num1 = 10
     var num2 = 20
     swapValues(&num1, &num2)
    print("num1 = \(num1), num2 = \(num2)")
       
     输出： num1 = 20, num2 = 10 
        
~~~~

注意点:

- 可变参数不能标记为inout
- inout参数不能有默认值
- inout参数的本质是地址传递(引用传递)
- inout参数只能传入可以被多次赋值的

## 准备代码

`inout` 是地址传递，对于不同的情况具体怎么传递呢？汇编拨开云雾

如下代码，表示等边的多边形，其中`width`表示边长，`side`表示多边形边长数量 `girth`表示周长，我们知道 `周长 = 边长 * 边数`

~~~~
struct Shape {
    // 宽、边长
    var width: Int
    // 边的数量
    var side: Int {
        willSet {
            print("willSetSide", newValue)
        }
        didSet {
            print("didSetSide", oldValue, side)
        }
    }
    // 周长
    var girth: Int {
        set {
            // 边长 = 周长 / 边数
            width = newValue / side
            print("setGirth", newValue)
        }
        get {
            print("getGirth")
            // 周长 = 边长 * 边数
            return width * side
        }
    }
    func show() {
        print("width=\(width), side=\(side), girth=\(girth)")
        
    }
}


func test(_ num: inout Int) {
 	print("test");
    num = 8
}
~~~~


## inout 修改存储属性
### 先看打印结果

 如下代码

~~~~
struct Shape {
    // 宽、边长
    var width: Int
    // 边的数量
    var side: Int {
        willSet {
            print("willSetSide", newValue)
        }
        didSet {
            print("didSetSide", oldValue, side)
        }
    }
    // 周长
    var girth: Int {
        set {
            // 边长 = 周长 / 边数
            width = newValue / side
            print("setGirth", newValue)
        }
        get {
            print("getGirth")
            // 周长 = 边长 * 边数
            return width * side
        }
    }
    func show() {
        print("width=\(width), side=\(side), girth=\(girth)")
        
    }
}


func test(_ num: inout Int) {
 	print("test");
    num = 8
}

var s = Shape(width: 10, side: 4)
test(&s.width) //这里打断点
s.show()
~~~~

打印结果为

~~~~
test
getGirth
width=8, side=4, girth=32
~~~~

其中`getGirth`这句打印是因为后面的`show`方法需要获取`girth`的值，如果我们去掉最后一句,只有如下代码

~~~~
var s = Shape(width: 10, side: 4)
test(&s.width)
~~~~

输出为

~~~~
test
~~~~

### 汇编分析

在上面代码中的 `test(&s.width)`这一句打断点

![](https://user-gold-cdn.xitu.io/2019/7/17/16bfcef672deebf0?w=1172&h=650&f=png&s=295179)


关键代码为

~~~~
//全局变量0x44be(%rip)的地址给寄存器rdi，rdi是全局变量s的地址
 0x100000fbb <+107>: leaq   0x44be(%rip), %rdi        ; testSwift.s : testSwift.Shape

// 调用test函数，其中rdi作为参数传入
0x100000fc2 <+114>: callq  0x100001930               ; testSwift.test(inout Swift.Int) -> () at main.swift:44
~~~~

### 小结 

 **把属性`s.width `的地址值传递过去，进行修改**
 
- 全局变量0x44be(%rip)的地址给寄存器rdi，rdi是全局变量s的地址
- 调用test函数，其中rdi作为参数传入


- 为什么我们代码中写的是 s.width ，但汇编传入的是s的地址呢？
    - 因为，width作为结构体的第一个属性变量，它的地址就是结构体s的地址

- 为什么rdi是作为参数呢？
	- [汇编总结](https://juejin.im/post/5d19f9816fb9a07f0a2df848)中我们知道 rdi、rsi、rdx、rcx、r8、r9等寄存器常用于存放函数参数。



## inout 修改带有属性观察器的存储属性


### 分析

首先分析一下，应该和前面存储属性不一样的，因为如果直接修改存储属性`side `的值，那怎么调动属性观察器的方法`willSet `和`didSet `呢？，

~~~~

struct Shape {
    // 宽、边长
    var width: Int
    // 边的数量
    var side: Int {
        willSet {
            print("willSetSide", newValue)
        }
        didSet {
            print("didSetSide", oldValue, side)
        }
    }
    // 周长
    var girth: Int {
        set {
            // 边长 = 周长 / 边数
            width = newValue / side
            print("setGirth", newValue)
        }
        get {
            print("getGirth")
            // 周长 = 边长 * 边数
            return width * side
        }
    }
    func show() {
        print("width=\(width), side=\(side), girth=\(girth)")
        
    }
}
func test(_ num: inout Int) {
    print("test");
    num = 8
}
var s = Shape(width: 10, side: 4)

test(&s.side) //这里打断点
~~~~

输出

~~~~
test
willSetSide 8
didSetSide 4 8
~~~~



### 查看汇编

![](https://user-gold-cdn.xitu.io/2019/7/17/16bfcef673dc1585?w=2092&h=1244&f=png&s=814277)

关键汇编代码分析

~~~~
//全局变量0x44e4(%rip)的地址给寄存器rdi，地址是testSwift.Shape + 8也就是size的地址
0x100000f9d <+109>: movq   0x44e4(%rip), %rax        ; testSwift.s : testSwift.Shape + 8

//size的地址给局部变量-0x28(%rbp)
0x100000fa4 <+116>: movq   %rax, -0x28(%rbp)

//局部变量-0x28(%rbp)的值给寄存器 %rdi
0x100000fa8 <+120>: leaq   -0x28(%rbp), %rdi

//调用test函数，寄存器 %rdi作为参数传入
0x100000fac <+124>: callq  0x100001930               ; testSwift.test(inout Swift.Int) -> () at main.swift:44

//此时已经修改完了局部变量 -0x28(%rbp)对应的值 并把局部变量 -0x28(%rbp)的值传给rdi,
0x100000fb1 <+129>: movq   -0x28(%rbp), %rdi
0x100000fb5 <+133>: leaq   0x44c4(%rip), %r13        ; testSwift.s : testSwift.Shape


//从截图中也可以看到此时%rdi里面是8，也就是 test函数中的  num = 8
0x100000fbc <+140>: callq  0x100001240               ; testSwift.Shape.side.setter : Swift.Int at <compiler-generated>
~~~~

### `testSwift.Shape.side.setter `函数中，调用`side.willset` 和 `side.didset`

![](https://user-gold-cdn.xitu.io/2019/7/17/16bfcef675ec0101?w=1988&h=1154&f=png&s=538327)


### 小结

对于带有属性观察器的存储属性`size`

- 首先把`size`的地址放在一个局部变量中
- 然后调用`test`方法，把局部变量的值修改
- 再把局部变量传入到`setter`方法中，真正的修改计算属性`size `

## inout 修改计算属性`girth `

### 分析

首先分析一下，应该和前面存储属性不一样的，因为计算属性`girth `没有自己的内存地址，

~~~~

struct Shape {
    // 宽、边长
    var width: Int
    // 边的数量
    var side: Int {
        willSet {
            print("willSetSide", newValue)
        }
        didSet {
            print("didSetSide", oldValue, side)
        }
    }
    // 周长
    var girth: Int {
        set {
            // 边长 = 周长 / 边数
            width = newValue / side
            print("setGirth", newValue)
        }
        get {
            print("getGirth")
            // 周长 = 边长 * 边数
            return width * side
        }
    }
    func show() {
        print("width=\(width), side=\(side), girth=\(girth)")
        
    }
}
func test(_ num: inout Int) {
    print("test");
    num = 8
}
var s = Shape(width: 10, side: 4)

test(&s.girth) //这里打断点
~~~~

输出

~~~~
getGirth
test
setGirth 8
~~~~



### 查看汇编

![](https://user-gold-cdn.xitu.io/2019/7/17/16bfcef676ac3806?w=2008&h=1198&f=png&s=568740)

### 汇编分析

关键汇编代码分析

~~~~	

// 调用testSwift.Shape.girth.getter 方法，返回值放在rax中
0x100000fab <+123>: callq  0x100001580               ; testSwift.Shape.girth.getter : Swift.Int at main.swift:33

// 把getter的返回值放在 局部变量-0x28(%rbp)中
0x100000fb0 <+128>: movq   %rax, -0x28(%rbp)

//局部变量-0x28(%rbp)的地址值 放在寄存器rdi
0x100000fb4 <+132>: leaq   -0x28(%rbp), %rdi

//寄存器rdi的地址值传到testSwift.test函数中，进行修改
0x100000fb8 <+136>: callq  0x100001930               ; testSwift.test(inout Swift.Int) -> () at main.swift:44

// 局部变量-0x28(%rbp)的值，传到寄存器rdi中
0x100000fbd <+141>: movq   -0x28(%rbp), %rdi


0x100000fc1 <+145>: leaq   0x44b8(%rip), %r13        ; testSwift.s : testSwift.Shape

// 寄存器rdi里面放的是局部变量-0x28(%rbp)的值 传入到Shape.girth.setter中
0x100000fc8 <+152>: callq  0x1000012f0               ;  testSwift.Shape.girth.setter : Swift.Int at main.swift:28
~~~~


### 小结
因为计算属性本身没有地址值，所以过程略显复杂

对于`inout`修改计算属性`girth`

- 首先调用`getter`方法，把返回值放在一个局部变量中
- 然后调用`test`方法，把局部变量的值修改
- 再把局部变量传入到`setter`方法中，真正的修改计算属性`girth`



## 总结

### 针对本文代码的总结
输入输出参数inout 本质就是引用传递，也就是地址传递，根据传过来的地址，修改对应的值。针对不同的情况，其他处理不同，

- 普通存储属性，直接把地址值传过来修改就可以了。
- 对于`inout`带有属性观察器的存储属性`size`
	- 首先把`size`的地址放在一个局部变量中
	- 然后调用`test`方法，把局部变量的值修改
	- 再把局部变量传入到`setter`方法中，真正的修改计算属性`size `
- 对于`inout`修改计算属性`girth`
	- 首先调用`getter`方法，把返回值放在一个局部变量中
	- 然后调用`test`方法，把局部变量的值修改
	- 再把局部变量传入到`setter`方法中，真正的修改计算属性`girth`

### 针对inout的总结

- 如果实参有物理内存地址，且没有设置属性观察器	
	- 直接将实参的内存地址传入函数（实参进行引用传递）
	- 如果实参是计算属性 或者 设置了属性观察器
- 采取了Copy In Copy Out的做法
	- 调用该函数时，先复制实参的值，产生副本【get】
	- 将副本的内存地址传入函数（副本进行引用传递），在函数内部可以修改副本的值
	- 函数返回后，再将副本的值覆盖实参的值【set】
- 总结：inout的本质就是引用传递（地址传递）

参考资料:


[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)


