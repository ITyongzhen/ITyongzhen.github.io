---
layout: post
title: Swift之类
date: 2019-07-08 17:32:24.000000000 +09:00
categories: 
- Swift
---

文章首发于[我的个人博客](https://ityongzhen.github.io/Swift之类.html)

## 前言

类的定义和结构体类似，但编译器并没有为类自动生成可以传入成员值的初始化器
eg 如下代码不会报错

~~~~
struct Point {
    var x: Int = 0
	var y: Int = 0 
}
let p1 = Point()
let p2 = Point(x: 10, y: 20)
let p3 = Point(x: 10)
let p4 = Point(y: 20)
~~~~

但是如果改成了类就不能编译通过

~~~~
class Point {
    var x: Int = 0
    var y: Int = 0
    
}
let p1 = Point()
let p2 = Point(x: 10, y: 20)//报错Argument passed to call that takes no arguments
let p3 = Point(x: 10)//报错Argument passed to call that takes no arguments
let p4 = Point(y: 20)//报错Argument passed to call that takes no arguments
~~~~


## 类的初始化器
- 如果类的所有成员都在定义的时候指定了初始值，编译器会为类生成无参的初始化器  
- 成员的初始化是在这个初始化器中完成的

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

## 结构体与类的本质区别
### 结构体是值类型(枚举也是值类型)，类是引用类型(指针类型)

eg:我们有如下的结构体point 和类 size

~~~~
// 类size
class Size {
    var width = 1
    var height = 2
}

// 类 Point
struct Point {
    var x = 3
    var y = 4
}

// 变量 size 接收类Size
var size = Size()
// 变量 point 接收结构体point
var point = Point()

~~~~

我们假设 执行完test() 之后，point的内存地址为 0x10000 size的内存地址为 0x10010
可以用一幅图来表示

![](https://user-gold-cdn.xitu.io/2019/7/7/16bca935cf3172a9?w=923&h=410&f=png&s=139924)

上图表示，point是值拷贝，直接把  3 和 4 放在了point对应的内存中，而 指针变量size是引用拷贝，是放了 Size() 的指针 0x90000 ，而对应的 堆空间 0x90000中才真正的存放1和2，当然了，前面有16个字节，存放了类的信息，和引用计数，因为Swift和OC一样使用的引用计数来内存管理的，所以Size对象用了32个字节

### 汇编验证

代码如下

~~~~
unc test1(){
    // 类size
    class Size {
        var width = 1
        var height = 2
    }
    
    // 类 Point
    struct Point {
        var x = 3
        var y = 4
    }
    
   
    // 变量 point 接收结构体point
    var point = Point()
     // 变量 size 接收类Size
    var size = Size()
}

test1()
~~~~

#### 汇编验证验证结构体

上面的代码，先在 var point = Point() 处打断点



~~~~
testSwift`__allocating_init() in Size #1 in test1():
    0x100001030 <+0>:  pushq  %rbp
    0x100001031 <+1>:  movq   %rsp, %rbp
    0x100001034 <+4>:  pushq  %r13
    0x100001036 <+6>:  subq   $0x18, %rsp
    0x10000103a <+10>: xorl   %eax, %eax
    0x10000103c <+12>: movl   %eax, %edi
->  0x10000103e <+14>: callq  0x100001250               ; type metadata accessor for Size #1 in testSwift.test1() -> () at <compiler-generated>
    0x100001043 <+19>: movl   $0x20, %ecx
    0x100001048 <+24>: movl   %ecx, %esi
    0x10000104a <+26>: movl   $0x7, %ecx
    0x10000104f <+31>: movl   %ecx, %edi
    0x100001051 <+33>: movq   %rdi, -0x10(%rbp)
    0x100001055 <+37>: movq   %rax, %rdi
    0x100001058 <+40>: movq   -0x10(%rbp), %rax
    0x10000105c <+44>: movq   %rdx, -0x18(%rbp)
    0x100001060 <+48>: movq   %rax, %rdx
    0x100001063 <+51>: callq  0x100005046               ; symbol stub for: swift_allocObject
    0x100001068 <+56>: movq   %rax, %r13
    0x10000106b <+59>: callq  0x100001e30               ; init() -> Size #1 in testSwift.test1() -> () in Size #1 in testSwift.test1() -> () at main.swift:15
    0x100001070 <+64>: addq   $0x18, %rsp
    0x100001074 <+68>: popq   %r13
    0x100001076 <+70>: popq   %rbp
    0x100001077 <+71>: retq   

~~~~

从 

~~~~
0x10000103e <+14>: callq  0x100001250 ; type metadata accessor for Size #1 in testSwift.test1() -> () at <compiler-generated> 
~~~~

处执行lldb命令 si 跟踪进去


~~~~

testSwift`init() in Point #1 in test1():
->  0x100001030 <+0>:  pushq  %rbp
    0x100001031 <+1>:  movq   %rsp, %rbp
    0x100001034 <+4>:  xorps  %xmm0, %xmm0
    0x100001037 <+7>:  movaps %xmm0, -0x10(%rbp)
    0x10000103b <+11>: movq   $0x3, -0x10(%rbp)
    0x100001043 <+19>: movq   $0x4, -0x8(%rbp)
    0x10000104b <+27>: movl   $0x3, %eax
    0x100001050 <+32>: movl   $0x4, %ecx
    0x100001055 <+37>: movl   %ecx, %edx
    0x100001057 <+39>: popq   %rbp
    0x100001058 <+40>: retq   

~~~~

可以看到赋值操作 直接是把 $0x3 和 $0x4 赋值给栈空间 (-0x10(%rbp) 和 -0x8(%rbp) )的，没有调用malloc alloc 等方法，也就是没有开辟堆空间

#### 汇编验证验证类

上面的代码，先在 var size = Size() 处打断点

~~~~
 0x100000fe0 <+0>:  pushq  %rbp
    0x100000fe1 <+1>:  movq   %rsp, %rbp
    0x100000fe4 <+4>:  pushq  %r13
    0x100000fe6 <+6>:  subq   $0x28, %rsp
    0x100000fea <+10>: movq   $0x0, -0x10(%rbp)
    0x100000ff2 <+18>: xorps  %xmm0, %xmm0
    0x100000ff5 <+21>: movaps %xmm0, -0x20(%rbp)
    0x100000ff9 <+25>: xorl   %eax, %eax
    0x100000ffb <+27>: movl   %eax, %edi
->  0x100000ffd <+29>: callq  0x100001250               ; type metadata accessor for Size #1 in testSwift.test1() -> () at <compiler-generated>
    0x100001002 <+34>: movq   %rax, %r13
    0x100001005 <+37>: movq   %rdx, -0x28(%rbp)
    0x100001009 <+41>: callq  0x100001030               ; __allocating_init() -> Size #1 in testSwift.test1() -> () in Size #1 in testSwift.test1() -> () at main.swift:15
    0x10000100e <+46>: movq   %rax, -0x10(%rbp)
    0x100001012 <+50>: callq  0x100001080               ; init() -> Point #1 in testSwift.test1() -> () in Point #1 in testSwift.test1() -> () at main.swift:21
    0x100001017 <+55>: movq   %rax, -0x20(%rbp)
    0x10000101b <+59>: movq   %rdx, -0x18(%rbp)
    0x10000101f <+63>: movq   -0x10(%rbp), %rdi
    0x100001023 <+67>: callq  0x1000050ac               ; symbol stub for: swift_release
    0x100001028 <+72>: addq   $0x28, %rsp
    0x10000102c <+76>: popq   %r13
    0x10000102e <+78>: popq   %rbp
    0x10000102f <+79>: retq   
~~~~

进入 

~~~~
 0x100001009 <+41>: callq  0x100001030               ; __allocating_init() -> Size #1 in testSwift.test1() -> () in Size #1 in testSwift.test1() -> () at main.swift:15
~~~~

一路跟踪进入，最终来到了如下图所示位置

![](https://user-gold-cdn.xitu.io/2019/7/7/16bca935cf4541ea?w=812&h=499&f=png&s=235950)

也就是确实分配了堆空间，验证了我们前面的结论

## 对象的堆空间申请过程

- 在Swift中，创建类的实例对象，要向堆空间申请内存，大概流程如下 

~~~~
Class.__allocating_init() 
libswiftCore.dylib:_swift_allocObject_ 
libswiftCore.dylib:swift_slowAlloc 
libsystem_malloc.dylib:malloc
~~~~

- 在Mac、iOS中的malloc函数分配的内存大小总是16的倍数
- 通过class_getInstanceSize可以得知:类的对象至少需要占用多少内存
 
eg:

~~~~
class Point  {
    var x = 11
    var test = true
	var y = 22 
}
var p = Point() 
class_getInstanceSize(type(of: p)) // 40
class_getInstanceSize(Point.self) // 40
~~~~

## 值类型

- 值类型赋值给var、let或者给函数传参，是直接将所有内容拷贝一份 
- 类似于对文件进行copy、paste操作，产生了全新的文件副本。属于深拷贝(deep copy)

- 在Swift标准库中，为了提升性能，String、Array、Dictionary、Set采取了Copy On Write的技术 
	- 比如仅当有“写”操作时，才会真正执行拷贝操作
 	- 对于标准库值类型的赋值操作，Swift 能确保最佳性能，所有没必要为了保证最佳性能来避免赋值
- 建议:不需要修改的，尽量定义成let


## 引用类型

- 引用赋值给var、let或者给函数传参，是将内存地址拷贝一份 
- 类似于制作一个文件的替身(快捷方式、链接)，指向的是同一个文件。属于浅拷贝(shallow copy)

## 枚举、结构体、类都可以定义方法

### 一般把定义在枚举、结构体、类内部的函数，叫做方法

~~~~
// 类中定义方法
class Size {
    var width = 10
    var height = 10
    func show() {
        print("width=\(width), height=\(height)")
    }
}
let s = Size()
s.show() // width=10, height=10

// 结构体中定义方法
struct Point {
    var x = 10
    var y = 10
    func show() {
        print("x=\(x), y=\(y)")
    }
}
let p = Point()
p.show() // x=10, y=10

// 枚举中定义方法
enum grade : Character {
    case a = "a"
    case b = "b"
    func show() {
        print("res is \(rawValue)")
    }
}
let g = grade.a
g.show() // res is a

~~~~

参考资料:


[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

[窥探内存细节的小工具](https://github.com/CoderMJLee/Mems)

