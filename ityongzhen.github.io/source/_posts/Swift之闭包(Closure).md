---
layout: post
title: Swift之闭包(Closure)
date: 2019-07-25 17:32:24.000000000 +09:00
categories: 
- Swift
---



本文首发于[我的个人博客](https://ityongzhen.github.io/Swift%E4%B9%8B%E9%97%AD%E5%8C%85(Closure).html)

## 什么是闭包

- 一个函数和它所捕获的变量\常量环境组合起来，称为闭包
	- 一般指定义在函数内部的函数
	- 一般它捕获的是外层函数的局部变量\常量
- 可以把闭包想象成是一个类的实例对象
	- 内存在堆空间
	- 捕获的局部变量\常量就是对象的成员(存储属性) 
	- 组成闭包的函数就是类内部定义的方法

eg: 
我们有一个函数 sum

~~~~
// 函数
func sum(_ v1: Int, _ v2: Int) -> Int { v1 + v2 }
// 使用
sum(10, 20)
~~~~

如果用闭包表达式定义一个函数

~~~~
var fn = {
    (v1: Int, v2: Int) -> Int in
    return v1 + v2
}

// 使用
fn(10, 20)
~~~~

当然了，也可以

~~~~
{
    (v1: Int, v2: Int) -> Int in
    return v1 + v2
}(10, 20)

~~~~

总结起来就是

~~~~
{
    (参数列表) -> 返回值类型 in 函数体代码
}
~~~~

## 闭包表达式的简写

我们定义如下的函数 exec ，它接收三个参数，分别为两个Int 和一个函数，而且这个函数，接收两个Int 参数，返回一个Int结果，exec 的作用就是，把前两个参数传给第三个参数(也就是函数)去执行，然后结果打印出来

~~~~

// 函数 我们定义如下的函数 exec ，它接收三个参数，分别为两个Int 和一个函数，而且这个函数，接收两个Int 参数，返回一个Int结果，exec 的作用就是，把前两个参数传给第三个参数(也就是函数)去执行，然后结果打印出来
func exec(v1: Int, v2: Int, fn: (Int, Int) -> Int) {
    print(fn(v1, v2))
}
~~~~

如果用闭包表达式来定义的话

~~~~
	// 闭包表达式
exec(v1: 10, v2: 20, fn: {
    (v1: Int, v2: Int) -> Int in
    return v1 + v2
})
~~~~

当然了，我们可以省略很多，如下

~~~~

// 省略参数类型 因为swift可以自己推断类型
exec(v1: 10, v2: 20, fn: {
    v1, v2 in return v1 + v2
})

// return 也可以省略
exec(v1: 10, v2: 20, fn: {
    v1, v2 in v1 + v2
})

// 省略掉参数列表，用$0代表第0个参数，$1代表第1个参数
exec(v1: 10, v2: 20, fn: {
    $0 + $1
})

// 终极省略
exec(v1: 10, v2: 20, fn: +)
~~~~
	
## 尾随闭包

- 如果将一个很长的闭包表达式作为函数的最后一个实参，使用尾随闭包可以增强函数的可读性
	- 尾随闭包是一个被书写在函数调用括号外面(后面)的闭包表达式

有如下的函数 闭包表达式作为函数的最后一个实参

~~~~	
func exec(v1: Int, v2: Int, fn: (Int, Int) -> Int) {
    print(fn(v1, v2))
}
	
~~~~


使用尾随闭包为



~~~~	
exec(v1: 10, v2: 20) {
    $0 + $1
}
~~~~


- 如果闭包表达式是函数的唯一实参，而且使用了尾随闭包的语法，那就不需要在函数名后边写圆括号

~~~~

// 这个闭包表达式表达式是函数的唯一实参
func exec(fn: (Int, Int) -> Int) { 
	print(fn(1, 2))
}

~~~~

可以使用尾随闭包如下

~~~~

// 使用尾随闭包如下三种都可以
exec(fn: { $0 + $1 })
exec() { $0 + $1 }
exec { $0 + $1 }

~~~~


## 尾随闭包实战

### 系统自带的排序
假设我们有个包含Int元素的数组，想对立面的元素进行排序

~~~~

func numberSort()  {
    var arr = [6, 8, 1, 10]
    arr.sort()
    print(arr) //[1, 6, 8, 10]
}

numberSort()

~~~~

打印结果为
>[1, 6, 8, 10]

查看官方对sort的源码为

~~~~

// 官方代码
func sort(by areInIncreasingOrder: (Element, Element) -> Bool)
~~~~

### 自定义排序

假如我们想自定义排序

~~~~

/// 返回true: i1排在i2前面
/// 返回false: i1排在i2后面
func cmp(i1: Int, i2: Int) -> Bool {
    // 大的排在前面
    return i1 > i2
}
~~~~

使用的时候

~~~~

var nums = [6, 8, 1, 10]
nums.sort(by: cmp)
print(nums)
~~~~

打印结果为
>[10, 8, 6, 1]

### 用尾随闭包书写

上面的代码 

可以写成

~~~~

nums.sort(by: {
    (i1: Int, i2: Int) -> Bool in
    return i1 > i2
})
~~~~

也可以等价于下面几种

~~~~

nums.sort(by: { i1, i2 in return i1 > i2 })
nums.sort(by: { i1, i2 in i1 > i2 })
nums.sort(by: { $0 > $1 })
nums.sort(by: > )
nums.sort() { $0 > $1 }
nums.sort { $0 > $1 }
~~~~

## 忽略参数

Swift中，很多时候，如果我们对于参数不做处理，可以用 下划线 _ 来代替

例如下面的闭包

~~~~

func exec(fn: (Int, Int) -> Int) {
    print(fn(1, 2))
}

print(exec{_,_ in 100 })  // 100
~~~~

输出  
>100


## 自动闭包

### 函数

假设我们定义一个这样的函数，要求 如果第1个数大于0，返回第一个数。否则返回第2个数

~~~~

func getFirstPositive(_ v1: Int, _ v2:  Int) -> Int? {
    return v1 > 0 ? v1 : v2
    
}
    
//调用
getFirstPositive(10, 20) // 10
getFirstPositive(-2, 20) // 20
getFirstPositive(0, -4) // -4
~~~~

现在假如说，我们这么传入的话

~~~~
func getNum() -> Int {
	// 这里每次都执行
    let a = 100
    let b = 200
    return a + b
}

func getFirstPositive2(_ v1: Int, _ v2:  Int) -> Int? {
    return v1 > 0 ? v1 : v2
    
}

getFirstPositive2(10, getNum())

~~~~

### 改成函数类型的参数

因为第一个参数已经是10 大于0了，第二个参数，也就是getNum() 根本没必要去执行，浪费性能，所以，有没有什么办法能做到，当第一个参数不满足时候，才去执行getNum()呢？答案是有的

~~~~
// 改成函数类型的参数，可以让v2延迟加载
func getFirstPositive2(_ v1: Int, _ v2: () -> Int) -> Int? {
	// 这里判断 v1 > 0 不会调用 v2()
    return v1 > 0 ? v1 : v2()
}

getFirstPositive2(10, {
	// 第一个参数大于0的时候，这里不会执行
    let a = 100
    let b = 200
    return a + b
})

~~~~


### 改进

如果改成这样写就报错了

~~~~
/ 改成函数类型的参数，可以让v2延迟加载
func getFirstPositive2(_ v1: Int, _ v2: () -> Int) -> Int? {
	// 这里判断 v1 > 0 不会调用 v2()
    return v1 > 0 ? v1 : v2()
}

getFirstPositive2(10,  20) //报错 Cannot convert value of type 'Int' to expected argument type '() -> Int'
~~~~

因为需要的是() -> Int类型，给的是Int

我们可以写成下面两种都可以

~~~~
/ 改成函数类型的参数，可以让v2延迟加载
func getFirstPositive2(_ v1: Int, _ v2: () -> Int) -> Int? {
	// 这里判断 v1 > 0 不会调用 v2()
    return v1 > 0 ? v1 : v2()
}

getFirstPositive2(10) { 20}

getFirstPositive2(10, {20})
~~~~

### @autoclosure

上面的也可以用自动闭包技术

~~~~
func getFirstPositive3(_ v1: Int, _ v2: @autoclosure () -> Int) -> Int? {
    return v1 > 0 ? v1 : v2()
}
getFirstPositive3(-4, 20)
~~~~

### 需要的注意点：
- 为了避免与期望冲突，使用了@autoclosure的地方最好明确注释清楚:这个值会被推迟执行
- @autoclosure 会自动将 20 封装成闭包 { 20 }
- @autoclosure 只支持 () -> T 格式的参数 n@autoclosure 并非只支持最后1个参数
- 空合并运算符 ?? 使用了 @autoclosure 技术
- 有@autoclosure、无@autoclosure，构成了函数重载



[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

