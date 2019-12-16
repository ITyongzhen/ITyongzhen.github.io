---
layout: post
title: Swift之结构体
date: 2019-07-16 17:32:24.000000000 +09:00
categories: 
- Swift
---



本文首发于[我的个人博客](https://ityongzhen.github.io/Swift之结构体.html)

## 前言

- 在 Swift 标准库中，绝大多数的公开类型都是结构体，而枚举和类只占很小一部分 
	- 比如Bool、Int、Double、 String、Array、Dictionary等常见类型都是结构体

~~~~
 struct Date {
		var year: Int		
		var month: Int
		var day: Int
	}
 var date = Date(year: 2019, month: 6, day: 23)
~~~~


- 所有的结构体都有一个编译器自动生成的初始化器(initializer，初始化方法、构造器、构造方法) 
	- 在最后一行调用的，可以传入所有成员值，用以初始化所有成员(存储属性，Stored Property)
 
## 结构体的初始化器

- 编译器会根据情况，可能会为结构体生成多个初始化器，宗旨是:保证所有成员都有初始值
     
eg:

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

如果给定一个初始值

~~~~

struct Point{
    var x: Int = 10
    var y: Int
}

var p1 = Point(x: 10, y: 20)
var p2 = Point(y: 20)
var p3 = Point(x: 10) //报错 Missing argument for parameter 'y' in call
var p4 = Point()    //报错 Missing argument for parameter 'y' in call
~~~~

如果x 和 y都有初始值的话，就怎么都不会报错了，因为 所有成员都有初始值

~~~~

struct Point{
    var x: Int = 10
    var y: Int = 20
}

var p1 = Point(x: 10, y: 20)
var p2 = Point(y: 20)
var p3 = Point(x: 10) 
var p4 = Point()    
~~~~

初始值为nil的话，也可以编译通过，比如下面这种

~~~~

struct Point{
    var x: Int?
    var y: Int?
}

var p1 = Point(x: 10, y: 20)
var p2 = Point(y: 20)
var p3 = Point(x: 10) 
var p4 = Point()    
~~~~

## 自定义初始化器

- 一旦在定义结构体时自定义了初始化器，编译器就不会再帮它自动生成其他初始化器
 
~~~~

struct Point{
    var x: Int = 10
    var y: Int = 20
    init(x: Int, y: Int) {
        self.x = x
        self.y = y
    }
}

var p1 = Point(x: 10, y: 20)
var p2 = Point(y: 20) //报错 Missing argument for parameter 'x' in call
var p3 = Point(x: 10) //报错 Missing argument for parameter 'y' in call
var p4 = Point()    //报错 Missing argument for parameter 'x' in call

~~~~

## 窥探初始化器的本质

### 以下2段代码完全等效

~~~~
struct Point {
    var x: Int = 0
	var y: Int = 0 
}
	
	
struct Point {
    var x: Int
    var y: Int
	init() { 
		x=0 
		y=0
	} 
}

~~~~

### 验证 

~~~~
func test(){
    struct Point {
        var x: Int = 0
        var y: Int = 0
    }
    let p = Point() //这一行打断点
    print(p)
}
test()
~~~~

和

~~~~
func test(){
    struct Point {
        var x: Int 
        var y: Int 
     	 init() {
           x=0
            y=0
       	}
    }
    let p = Point() //这一行打断点
    print(p)
}
test()
~~~~

查看汇编，两段代码都是

~~~~
testSwift`init() in Point #1 in test():
->  0x100001d50 <+0>:  pushq  %rbp
    0x100001d51 <+1>:  movq   %rsp, %rbp
    0x100001d54 <+4>:  xorps  %xmm0, %xmm0
    0x100001d57 <+7>:  movaps %xmm0, -0x10(%rbp)
    0x100001d5b <+11>: movq   $0x0, -0x10(%rbp)
    0x100001d63 <+19>: movq   $0x0, -0x8(%rbp)
    0x100001d6b <+27>: xorl   %eax, %eax
    0x100001d6d <+29>: movl   %eax, %ecx
    0x100001d6f <+31>: movq   %rcx, %rax
    0x100001d72 <+34>: movq   %rcx, %rdx
    0x100001d75 <+37>: popq   %rbp
    0x100001d76 <+38>: retq   
~~~~

这两段代码的汇编一样的，也就是说，这两段代码完全等效

## 结构体的内存结构

~~~~

 struct Point {
        var x: Int = 0
        var y: Int = 0
        var origin: Bool = true

    }
    
 print(MemoryLayout<Point>.size)
 print(MemoryLayout<Point>.stride)
 print(MemoryLayout<Point>.alignment)
~~~~

打印结果为

>17

>24

>8

是因为内存对齐的缘故，17是因为 实际使用的是 8+8+1 = 17
24 是因为，要内存对齐，8*3 = 24






参考资料:


[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

[窥探内存细节的小工具](https://github.com/CoderMJLee/Mems)

