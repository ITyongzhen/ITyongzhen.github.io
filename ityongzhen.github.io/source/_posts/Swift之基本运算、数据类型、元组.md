
---
layout: post
title: Swift之基本运算、数据类型、元组
date: 2019-05-15 17:32:24.000000000 +09:00
categories: 
- Swift
---



本文首发于[我的个人博客](https://ityongzhen.github.io/categories/Swift/)

## 引言 

>   Swift编程语言，支持多编程范式和编译式，用来撰写基于macOS/OS X、iOS、watchOS和tvOS的软件。 苹果公司于2014年在苹果开发者年会（WWDC）发布了Swift编程语言。从设计上苹果公司让Swift与Objective-C共存在苹果公司的操作系统上

>苹果宣称Swift的特点是：快速、现代、安全、互动，而且明显优于Objective-C语言。Swift以LLVM编译，可以使用现有的Cocoa和Cocoa Touch框架。Xcode Playgrounds功能是Swift为苹果开发工具带来的最大创新，该功能提供强大的互动效果，能让Swift源代码在撰写过程中能即时显示出其运行结果

>2015年6月8日，苹果于WWDC2015上宣布，Swift将开放源代码，包括编译器和标准库
>2015年12月3日，苹果宣布[开源Swift](https://github.com/apple/Swift)，并支持Linux，苹果在新网站Swift.org和托管网站Github上开源了Swift，但苹果的app store并不支持开源的Swift，只支持苹果官方的Swift版本，官方版本会在新网站Swift.org上定期与开源版本同步


之前由于每个版本都不兼容，所以对开发者不友好，每次新版本，就像学了一门新的语言。但是从Swift5开始，API终于稳定下来了。可以来总结，学习一下Swift了。

## 一些基本操作
- 生成语法树: Swiftc -dump-ast main.Swift 
- 生成最简洁的SIL代码:Swiftc -emit-sil main.Swift 
- 生成LLVM IR代码: Swiftc -emit-ir main.Swift -o main.ll 
- 生成汇编代码: Swiftc -emit-assembly main.Swift -o main.s

## 基础语法

- 不用编写main函数，Swift将全局范围内的首句可执行代码作为程序入口
- 一句代码尾部可以省略分号(;)
- 多句代码写到同一行时必须用分号(;)隔开 
-  用var定义变量，let定义常量，编译器能自动推断出变量\常量的类型
- Playground可以快速预览代码效果，是学习语法的好帮手 
 - Command + Shift + Enter: 运行整个Playground 
 -  Shift + Enter:运行截止到某一行代码

例如

~~~~
var a = 10
a = 20
let b = 88
let c = 12 ; let d = 40
print(d)
~~~~


创建对象，例如view视图，控制器等也更简单


## 注释
和OC一样，有单行注释，多行注释
例如

~~~~
// 单行注释

/*
 多行注释
 
 */
 
~~~~

但是，Swift中增加了，注释的嵌套，比如这样是可以的

~~~~
/*
 多行注释
 /*
 
 // 单行注释
 
 嵌套多行注释
 */
 
 */
~~~~

***Playound的注释是支持markup(类似Markdown)语法的***

+ 开启markup:Editor->Show Rendered Markup
+ 只能在Playground中使用

## 常量
- 只能赋值1次
- 常量的值不要求在编译的时候确定，只要在使用之前赋值一次就可以了
例如下面都是可以的

~~~~
var num = 10
num += 20
num += 30
let age = num
print(age)

func getAge() -> Int {
    return 10
}
let age2 = getAge()
print(age2)

~~~~

但是下面这种是不可以的，因为在初始化之前，是不可以使用的

~~~~
let number: Int
print(number)
~~~~

当然了，这种也是不行的

~~~~
let number
number = 25
~~~~

## 标识符
比如常量名，变量名，函数名等标识符

- 标识符不能数字开头，不能包含空白字符，制表符，箭头灯特殊字符。
- 除此之外，几乎可以使用任何字符。

## 常见数据类型

1. 值类型(value type)
	- 枚举(enum): optional
	- 结构体(struct): Float、 Double、Float、Int、Character、String、Array、Dictionary、Set
2. 引用类型(reference type)
	- 类(class)

## 类型转换
不同类型之间的转换，比如

~~~~
// 整数转换
let a: UInt16 = 2_000
let b: UInt8 = 10
let c = a + UInt16(b)
print(c)


//整数和浮点数转换
let intNumber = 3
let doubleNumber = 0.1415926
let pi = Double(intNumber) + doubleNumber
let intPi = Int(pi)

// 字面量可以直接相加，因为字面量没有明确的类型
let res = 3 + 2.565


~~~~


## 元组

元祖可以把多个值保存在一起

~~~~
格式: (数值1, 数值2, 数值3)
let numbers = (10, 20, 30)
// 可以通过索引访问
numbers.0
numbers.1
numbers.2
~~~~

元祖中还可以保存不同的数据类型的值

~~~~
let person = (name: "lnj", age: 30, score: 100.0)
// 可以通过名称访问
person.name
person.age
person.score
~~~~

我们甚至可以这样子

~~~~
// 相当于同时定义了三个变量
let (name, age, score) = ("lnj", 30, 80)
name
age
score

~~~~




参考资料:

[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

