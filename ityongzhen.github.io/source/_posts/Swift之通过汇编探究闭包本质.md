---
layout: post
title: Swift之通过汇编探究闭包本质
date: 2019-08-10 17:32:24.000000000 +09:00
categories: 
- Swift
---


本文首发于[我的个人博客](https://ityongzhen.github.io/Swift%E4%B9%8B%E9%80%9A%E8%BF%87%E6%B1%87%E7%BC%96%E6%8E%A2%E7%A9%B6%E9%97%AD%E5%8C%85%E6%9C%AC%E8%B4%A8.html)

## 前言

先回顾一下，上一篇  [Swift之闭包(Closure)](https://juejin.im/post/5d229738f265da1b715317b7)中对闭包的解释

- 一个函数和它所捕获的变量\常量环境组合起来，称为闭包
	- 一般指定义在函数内部的函数
	- 一般它捕获的是外层函数的局部变量\常量
- 可以把闭包想象成是一个类的实例对象
	- 内存在堆空间
	- 捕获的局部变量\常量就是对象的成员(存储属性) 
	- 组成闭包的函数就是类内部定义的方法


## 问题

先看下面一段代码，猜猜会输出什么

~~~~
typealias Fn = (Int) -> Int
func getFn() -> Fn{
    // 局部变量
    var num = 0
    func plus(_ i: Int) -> Int{
        num += i
        return num
    }
    return plus(_:)
}

var fn = getFn()
print(fn(1)) // 1
print(fn(2))	// 3
print(fn(3))	// 6
print(fn(4))	// 10
~~~~

结果是输出

~~~~
1
3
6
10
~~~~

那么，问题来了，为什么输出的是10呢？因为按照常识，var num = 0 是局部变量，执行完就销毁了，怎么能再后面继续使用呢？

## 验证

我们先从简单的说起
首先是下面一端代码

~~~~
typealias Fn = (Int) -> Int
func getFn() -> Fn{
    // 局部变量
    var num = 0
    func plus(_ i: Int) -> Int{
        return i 
    }
    return plus(_:) // 这里打断点
}

var fn = getFn()
print(fn(1))
~~~~

先不适用num ,直接 return i 并在这里打断点，结果如下

~~~~
testSwift`getFn():
    0x100001f70 <+0>:  pushq  %rbp
    0x100001f71 <+1>:  movq   %rsp, %rbp
    0x100001f74 <+4>:  movq   $0x0, -0x8(%rbp)
->  0x100001f7c <+12>: leaq   0xd(%rip), %rax           ; plus #1 (Swift.Int) -> Swift.Int in testSwift.getFn() -> (Swift.Int) -> Swift.Int at main.swift:23
    0x100001f83 <+19>: xorl   %ecx, %ecx
    0x100001f85 <+21>: movl   %ecx, %edx
    0x100001f87 <+23>: popq   %rbp
    0x100001f88 <+24>: retq   
~~~~

![](https://user-gold-cdn.xitu.io/2019/7/8/16bcf1fb505d92c2?w=1243&h=825&f=png&s=356842)

可知，0xd(%rip), %rax  这段代码，把地址值，也就是getFn() 函数的地址值给了rax,
根本没有alloc malloc等代码，也就是说，没有开辟堆空间。那么接下来我们看下面的代码

~~~~
typealias Fn = (Int) -> Int
func getFn() -> Fn{
    // 局部变量
    var num = 0
    func plus(_ i: Int) -> Int{
		 num += i
		 return num 
    }
    return plus(_:) // 这里打断点
}

var fn = getFn()
print(fn(1))
print(fn(2))
print(fn(3))
~~~~

断点如下

~~~~
testSwift`getFn():
    0x100001de0 <+0>:  pushq  %rbp
    0x100001de1 <+1>:  movq   %rsp, %rbp
    0x100001de4 <+4>:  subq   $0x20, %rsp
    0x100001de8 <+8>:  leaq   0x3301(%rip), %rdi
    0x100001def <+15>: movl   $0x18, %esi
    0x100001df4 <+20>: movl   $0x7, %edx
    
    // 这里swift_allocObject 说明产生了堆空间
    0x100001df9 <+25>: callq  0x1000046f8        ; symbol stub for: swift_allocObject
    0x100001dfe <+30>: movq   %rax, %rdx
    0x100001e01 <+33>: addq   $0x10, %rdx
    0x100001e05 <+37>: movq   %rdx, %rsi
    0x100001e08 <+40>: movq   $0x0, 0x10(%rax)
->  0x100001e10 <+48>: movq   %rax, %rdi
    0x100001e13 <+51>: movq   %rax, -0x8(%rbp)
    0x100001e17 <+55>: movq   %rdx, -0x10(%rbp)
    0x100001e1b <+59>: callq  0x100004758         ; symbol stub for: swift_retain
    0x100001e20 <+64>: movq   -0x8(%rbp), %rdi
    0x100001e24 <+68>: movq   %rax, -0x18(%rbp)
    0x100001e28 <+72>: callq  0x100004752         ; symbol stub for: swift_release
    0x100001e2d <+77>: movq   -0x10(%rbp), %rax
    0x100001e31 <+81>: leaq   0x178(%rip), %rax   ; partial apply forwarder for plus #1 (Swift.Int) -> Swift.Int in testSwift.getFn() -> (Swift.Int) -> Swift.Int at <compiler-generated>
    0x100001e38 <+88>: movq   -0x8(%rbp), %rdx
    0x100001e3c <+92>: addq   $0x20, %rsp
    0x100001e40 <+96>: popq   %rbp
    0x100001e41 <+97>: retq   
~~~~

进一步验证，下面的代码是因为，写文章的时候，重新跑了一遍，所以函数 getFn() 函数的抵制和截图不一致，是

> rax = 0x0000000101849fd0

这次我们在 

~~~~
typealias Fn = (Int) -> Int
func getFn() -> Fn{
    // 局部变量
    var num = 0
    func plus(_ i: Int) -> Int{
		 num += i
		 return num  // 第二次这里打断点 查看getFn()地址的内容
    }
    return plus(_:) // 第一次这里打断点 获取getFn()地址
}

var fn = getFn()
print(fn(1)) 
print(fn(2))
print(fn(3))
~~~~

因为调用了三次 fn分别为 fn(1) 、 fn(2)、fn(3)，所以在 return num 地方，会断三次
我们分别查看函数getFn() 函数地址的内容

结果如图
![](https://user-gold-cdn.xitu.io/2019/7/8/16bcf1fb506737cb?w=1139&h=886&f=png&s=362546)

图中可知，确实是操作同一块堆空间，而且之前[Swift之类](https://ityongzhen.github.io/Swift%E4%B9%8B%E7%B1%BB.html)中讲过，前面16个字节，分别存放 类的信息，引用技术，然后后面才是值，可知，

刚开始分配完，堆空间里面是垃圾数据
执行完 print(fn(1))  之后，堆空间里面放的是1
执行完 print(fn(2))  之后，堆空间里面放的是3
执行完 print(fn(3))  之后，堆空间里面放的是6

## 结论
这也解释了，文章开头的那个疑问，因为闭包捕获了局部变量，在堆中开辟空间，然后后面调用的时候，操作的是堆空间的内存，所以结果是

~~~~
1
3
6
10
~~~~

关于汇编的调试指令可以参考

[汇编总结](https://ityongzhen.github.io/%E6%B1%87%E7%BC%96%E6%80%BB%E7%BB%93.html)

[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

[汇编总结](https://ityongzhen.github.io/%E6%B1%87%E7%BC%96%E6%80%BB%E7%BB%93.html)

[Swift之闭包(Closure)](https://juejin.im/post/5d229738f265da1b715317b7)

