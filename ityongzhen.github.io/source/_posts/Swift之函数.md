---
layout: post
title: Swift之函数
date: 2019-06-10 17:32:24.000000000 +09:00
categories: 
- Swift
---

本文首发于[我的个人博客](https://ityongzhen.github.io/categories/Swift/)

## 函数定义
+ 形参默认是let 并且只能是let

1. 无参无返回值

 可以省略Void 也可以不省略，如下三种都可以
 
~~~~
func sayHello(){
    print("hello")
}

func sayHello() -> (){
    print("hello")
}

func sayHello() -> (Void){
    print("hello")
}
~~~~

2. 无参有返回值

~~~~
	 func pi() -> Double {
        return 3.1415
    }
  
~~~~

3. 有参有返回值
~~~~
    
    func sum(a: Int, b: Int) -> Int {
        return a + b
    }
~~~~

## 隐式返回
- 如果函数体是个单一表达式，那么函数会返回这个表达式
 比如上面的代码可以去掉 return 写成
 
~~~~
    func sum(a: Int, b: Int) -> Int {
        a + b
    }
~~~~

## 返回元组：可以实现多返回值

例如 

~~~~
  func calculate(a: Int, b: Int) -> (sum: Int, average: Int) {
        let sum = a + b
        return (sum, sum >> 1)
   }
   调用 calculate(a: 2, b: 8)
   返回 (sum: 10, average: 5)
~~~~


## 文档注释
- 把光标定位到需要添加注释文档的对象起始行，或上方的空白行。按下“command + Option + /”，即“⌘ + ⌥ + /”。（如果是 Windows 键盘则为“Win键 + ALT + /”）

例如上文代码增加注释

~~~~
 	/// 计算两个数之和
    ///
    /// - Parameters:
    ///   - a: 第一个参数
    ///   - b: 第二个参数
    /// - Returns: 返回两个参数之和
    func sum(a: Int, b: Int) -> Int {
        return a + b
    }
~~~~

## 参数标签
- 可以修改参数标签

~~~~
 func goToWork(at time: String) -> () {
        print("this time is \(time)")
    }
~~~~

- 可以使用下划线 _ 省略参数标签

~~~~
 func sum2(_ a: Int, _ b: Int) -> Int {
        return a + b
    }
~~~~
    
## 默认参数值
- 同C++ 中一样参数可以有默认值
- 但是C++里面默认参数有个限制：必须从右向左设置。而Swift中拥有参数标签，所以没有这个限制    

~~~~
 定义：func check(name: String = "jack", age: Int, job: String = "teacher") {
        print("name = \(name), age = \(age), job = \(job)")
    }
    
 调用：check(age: 22)
 
 输出： name = jack, age = 22, job = teacher
~~~~

## 可变参数
例如：

~~~~
 func sum(_ numbers: Int...) -> Int{
        var total = 0
        for num in numbers {
            total += num
        }
        return total
    }
	
	调用: sum(1,2,3)
~~~~

- 一个函数最多只能有一个可变参数
- 紧跟着可变参数后面的参数不能省略参数标签(否则编译起来有歧义)

例如

~~~~
// 参数string 不能省略标签
 func test(_ numers: Int..., string: String) -> Void {
        
    }

~~~~

## 输入输出参数

- 前面说了形参只能是let，但是如果我们想再内部修改外部实参的值，可以用 inpot 定义输入输出参数

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

- 可变参数不能标记为input
- input参数不能有默认值
- input参数的本质是地址传递(引用传递)
- input参数只能传入可以被多次赋值的

## 函数重载
### 规则
- 函数名相同
- 参数个数不同 或者 参数类型不同 或者 参数标签不同

注意点是：

- 返回值类型与函数重载无关
- 默认参数值和函数重载一起使用产生二义性时候，编译器不会报错(c++中会报错)

例如

~~~~
 	func sum(v1: Int, v2: Int) -> Int {
        v1 + v2
    }
    func sum(v1: Int, v2: Int, v3: Int = 10) -> Int {
        v1 + v2 + v3
        
    }
    
    // 会调用sum(v1: Int, v2: Int) 
    sum(v1: 10, v2: 20)
~~~~

- 可变参数、省略参数标签、函数重载一起使用产生二义性时，编译器有可能会报错

~~~~
    func sum(v1: Int, v2: Int) -> Int { v1 + v2
    }
    func sum(_ v1: Int, _ v2: Int) -> Int {
        v1 + v2 }
    func sum(_ numbers: Int...) -> Int { var total = 0
        for number in numbers {
            total += number
        }
        return total
    }
    // error: ambiguous use of 'sum'
    sum(10, 20)
~~~~

## 函数类型

- 每一个函数都是有类型的，函数类型由形式参数类型、返回值类型组成

~~~~
    func test() { } // () -> Void 或者 () -> ()
    func sum(a: Int, b: Int) -> Int {
        a+b
    } // (Int, Int) -> Int
    // 定义变量
    var fn: (Int, Int) -> Int = sum
    调用: fn(2, 3)
~~~~

## 函数类型作为函数参数
例如

~~~~
 func sum(v1: Int, v2: Int) -> Int {
        v1 + v2
    }
    func difference(v1: Int, v2: Int) -> Int {
        v1 - v2
        
    }
    // 用一个函数类型作为参数 上面两个函数类型都是 (Int, Int) -> Int
    func printResult(_ mathFn: (Int, Int) -> Int, _ a: Int, _ b: Int) { print("Result: \(mathFn(a, b))")
    }
    // 调用
    printResult(sum, 5, 2) // Result: 7
    printResult(difference, 5, 2) // Result: 3
~~~~

## 返回值是函数类型的函数
- 返回值是函数类型的函数，叫做高阶函数(Higher-Order Function)

~~~~
    func next(_ input: Int) -> Int {
        input + 1
    }
    
    func previous(_ input: Int) -> Int {
        input - 1 
    }
        
    func forward(_ forward: Bool) -> (Int) -> Int {
        forward ? next : previous
    }
    //调用
    forward(true)(3) //  4  相当于 调用 next(3)
    forward(false)(3) // 2 相当于 调用 previous(3)
~~~~

## typealias 别名
- 用来给类型起别名

~~~~
  typealias Date = (year: Int, month: Int, day: Int)
    func test(_ date: Date) {
        print(date.0)
        print(date.year)
        
    }
    // 调用
	// test((2011, 9, 10))
~~~~


## 嵌套函数
- 将函数定义在函数内部

~~~~
    func forward(_ forward: Bool) -> (Int) -> Int { func next(_ input: Int) -> Int {
        input + 1 }
        func previous(_ input: Int) -> Int { input - 1
        }
        return forward ? next : previous
    }
    
    forward(true)(3) // 4
    forward(false)(3) // 2
~~~~

参考资料:


[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

