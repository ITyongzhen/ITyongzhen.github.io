---
layout: post
title: Swift之枚举
date: 2019-06-20 17:32:24.000000000 +09:00
categories: 
- Swift
---



本文首发于[我的个人博客](https://ityongzhen.github.io/categories/Swift/)

## 枚举

### 枚举的基本用法
#### 定义
~~~~
//定义方向的枚举
enum Direction {
     case north
     case south
	 case east
	 case west 
}
~~~~
上面也可以写成

~~~~
enum Direction {
    case north, south, east, west
}
~~~~

#### 使用

~~~~
var dir = Direction.west 
dir = Direction.east 
dir = .north
print(dir) // north
~~~~

也可以在switch中使用

~~~~

switch dir { 
case .north:
	print("north") 
case .south:
    print("south") 
case .east:
	print("east") 
case .west:
    print("west")
}
~~~~

## 关联值

### 有时将枚举的成员值跟其他类型的值关联存储在一起，会非常有用，可以认为将值直接存入到枚举的内存中

~~~~

  enum Score {
    case points(Int)
    case grade(Character)
}
~~~~

如下使用：

~~~~

var score = Score.points(88) 
score = .grade("A")

~~~~

如果我们想使用枚举的具体值，可以如下用 i 来保存数据

~~~~

  switch score {
	case let .points(i):
		print(i, "points") 
	case let .grade(i):
    	print("grade", i)
} // grade A

~~~~


再比如我们想定义日期的枚举值，可以如下：

~~~~

enum Date {
    case digit(year: Int, month: Int, day: Int)
    case string(String)
}

//使用的时候，可以直接传年月日，或者传字符串

var date = Date.digit(year: 2011, month: 9, day: 10) 
date = .string("2011-09-10")
switch date {
	case .digit(let year, let month, let day):
			rint(year, month, day) 
	case let .string(value):
    	print(value)
}

~~~~

### 必要时let也可以改为var

## 原始值

### 枚举成员可以使用相同类型的默认值预先对应，这个默认值叫做:原始值

~~~~

// 定义枚举
enum Grade : String {
    case perfect = "A"
    case great = "B"
    case good = "C"
    case bad = "D"
}

// 使用
print(Grade.perfect.rawValue) // A 
print(Grade.great.rawValue) // B 
print(Grade.good.rawValue) // C 
print(Grade.bad.rawValue) // D

~~~~

### 注意:原始值不占用枚举变量的内存


## 隐式原始值(Implicitly Assigned Raw Values)
### 如果枚举的原始值类型是Int、String，Swift会自动分配原始值

### 原始值是 String 类型枚举值

~~~~

// 定义枚举值
  enum Direction : String {
    case north = "north"
    case south = "south"
    case east = "east"
    case west = "west"
}
// 等价于
enum Direction : String {
    case north, south, east, west
}
// 使用
print(Direction.north) // north 
print(Direction.north.rawValue) // north

~~~~

### 原始值是 Int 类型枚举值

~~~~

enum Season : Int {
    case spring, summer, autumn, winter
}
print(Season.spring.rawValue) // 0 
print(Season.summer.rawValue) // 1 
print(Season.autumn.rawValue) // 2 
print(Season.winter.rawValue) // 3

~~~~

### 如果自己指定了原始值

~~~~

enum Season : Int {
    case spring = 1, summer, autumn = 8, winter
}
print(Season.spring.rawValue) // 1 
print(Season.summer.rawValue) // 2 
print(Season.autumn.rawValue) // 8 
print(Season.winter.rawValue) // 9

~~~~

## 递归枚举

### 递归枚举要加上关键字 indirect

eg:

~~~~
// 递归枚举
indirect enum ArithExpr {
	case number(Int)
	case sum(ArithExpr, ArithExpr)
	case difference(ArithExpr, ArithExpr)
}

// 上面的递归枚举和下面的等效

enum ArithExpr {
	case number(Int)
	indirect case sum(ArithExpr, ArithExpr) 
	indirect case difference(ArithExpr, ArithExpr)
}

// 下列几种使用枚举都是可以的

let five = ArithExpr.number(5)
let four = ArithExpr.number(4)
let two = ArithExpr.number(2)
let sum = ArithExpr.sum(five, four)
let difference = ArithExpr.difference(sum, two)


// 自己写个 calculate 方法，
func calculate(_ expr: ArithExpr) -> Int { 
	switch expr {
	case let .number(value): 
		return value
	case let .sum(left, right):
		return calculate(left) + calculate(right)
	case let .difference(left, right):
		return calculate(left) - calculate(right)
} }

//最终调用，计算差值
calculate(difference)
~~~~

## MemoryLayout

### 可以使用MemoryLayout获取数据类型占用的内存大小

#### 关联值

- 将关联值直接存入到枚举内存中

~~~~
// 定义枚举
 enum Password {
    case number(Int, Int, Int, Int)
    case other
}

MemoryLayout<Password>.stride // 40, 分配占用的空间大小 
MemoryLayout<Password>.size // 33, 实际用到的空间大小 4*8 + 1 = 33
MemoryLayout<Password>.alignment // 8, 对齐参数
~~~~

定义变量来使用

~~~~

var pwd = Password.number(9, 8, 6, 4)

MemoryLayout.stride(ofValue: pwd) // 40
MemoryLayout.size(ofValue: pwd) // 33 
MemoryLayout.alignment(ofValue: pwd) // 8
 
// 如果改变了pwd
 pwd = .other 
MemoryLayout.stride(ofValue: pwd) // 40
MemoryLayout.size(ofValue: pwd) // 33 
MemoryLayout.alignment(ofValue: pwd) // 8

~~~~

#### 原始值 
- 原始值不会直接存入到枚举内存中
- 如果是下面这种枚举，只需要1个字节就可以了
- 一个字节可以存放 FF 也就是 0~255个枚举值。如果

~~~~
enum Season : Int {
    case spring, summer, autumn = 8, winter
}

MemoryLayout<Season>.stride // 1, 分配占用的空间大小 
MemoryLayout<Season>.size // 1, 实际用到的空间大小 1
MemoryLayout<Season>.alignment // 1, 对齐参数

~~~~

## 窥探内存

使用 [窥探内存细节的小工具](https://github.com/CoderMJLee/Mems) 我们可以很轻松的获取swift中，这些枚举值的内存地址

### 简单枚举内存

~~~~

enum TestEnum{
    case k0,k1,k2,k3
}

var t = TestEnum.k1
print(Mems.ptr(ofVal: &t)) 

t = TestEnum.k2

~~~~

执行完 print(Mems.ptr(ofVal: &t))  代码之后 
输出

~~~~
0x00000001000054b8
~~~~
此时去查看 0x00000001000054b8地址的数据，

~~~~
 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
~~~~
  
执行完 t = TestEnum.k2 之后

此时去查看 0x00000001000054b8地址的数据，

~~~~
 02 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
~~~~

### 具体查看内存

假设我们有这么下面带代码，考虑下内存怎么布局呢？

~~~~

enum TestEnum {
        case test1(Int, Int, Int)
        case test2(Int, Int)
        case test3(Int)
        case test4(Bool)
        case test5
}

print(MemoryLayout<TestEnum>.size)  // 25, 分配占用的空间大小
print(MemoryLayout<TestEnum>.stride)    //32, 实际用到的空间大小
print(MemoryLayout<TestEnum>.alignment)// 8, 对齐参数


var e = TestEnum.test1(1, 2, 3)
print(Mems.ptr(ofVal: &e))
  
  
e = .test2(4, 5)
print(Mems.memStr(ofVal: &e))
    
e = .test3(6)
     
e = .test4(true)
      
e = .test5
~~~~

执行完之后,可知 

TestEnum 这个占用内存为：

- print(MemoryLayout<TestEnum>.size)  // 25, 分配占用的空间大小
- print(MemoryLayout<TestEnum>.stride)    //32, 实际用到的空间大小
- print(MemoryLayout<TestEnum>.alignment)// 8, 对齐参数

具体内存里面存的是什么呢？可以借助上面说的 [窥探内存细节的小工具](https://github.com/CoderMJLee/Mems) 打印出来内存，然后利用Xcode的 view Memory 查看具体内存的值

结果如下


~~~~
 enum TestEnum {
        case test1(Int, Int, Int)
        case test2(Int, Int)
        case test3(Int)
        case test4(Bool)
        case test5
    }
    
    
    print(MemoryLayout<TestEnum>.size)  // 25, 分配占用的空间大小
    print(MemoryLayout<TestEnum>.stride)    //32, 实际用到的空间大小
    print(MemoryLayout<TestEnum>.alignment)// 8, 对齐参数

    // 1个字节存储成员值
    // N个字节存储关联值（N取占用内存最大的关联值），任何一个case的关联值都共用这N个字节
    // 共用体
    
    // 小端：高高低低
    // 01 00 00 00 00 00 00 00  //对应的TestEnum.test1传入的数值
    // 02 00 00 00 00 00 00 00
    // 03 00 00 00 00 00 00 00
    // 00                // TestEnum.test1在第0个位置
    // 00 00 00 00 00 00 00
    var e = TestEnum.test1(1, 2, 3)
    print(Mems.ptr(ofVal: &e))
    
    // 04 00 00 00 00 00 00 00
    // 05 00 00 00 00 00 00 00
    // 00 00 00 00 00 00 00 00
    // 01						 // TestEnum.test1在第1个位置
    // 00 00 00 00 00 00 00
    e = .test2(4, 5)
    print(Mems.memStr(ofVal: &e))
    
    // 06 00 00 00 00 00 00 00
    // 00 00 00 00 00 00 00 00
    // 00 00 00 00 00 00 00 00
    // 02
    // 00 00 00 00 00 00 00
    e = .test3(6)
    
    // 01 00 00 00 00 00 00 00
    // 00 00 00 00 00 00 00 00
    // 00 00 00 00 00 00 00 00
    // 03
    // 00 00 00 00 00 00 00
    e = .test4(true)
    
    // 00 00 00 00 00 00 00 00
    // 00 00 00 00 00 00 00 00
    // 00 00 00 00 00 00 00 00
    // 04
    // 00 00 00 00 00 00 00
    e = .test5
~~~~

### 只有一个case

假如只有一个case,其占用的空间为0，不需要存值来区分是哪个case

~~~~
enum TestEnum {
    case spring
}

print(MemoryLayout<TestEnum>.size)      // 0, 分配占用的空间大小
print(MemoryLayout<TestEnum>.stride)    //1, 实际用到的空间大小
print(MemoryLayout<TestEnum>.alignment)// 1, 对齐参数

~~~~

### 只有一个case，有关联值

~~~~
enum TestEnum {
    case spring(Int)
}

print(MemoryLayout<TestEnum>.size)      // 8, 分配占用的空间大小
print(MemoryLayout<TestEnum>.stride)    //8, 实际用到的空间大小
print(MemoryLayout<TestEnum>.alignment)// 8, 对齐参数

~~~~




参考资料:


[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

[窥探内存细节的小工具](https://github.com/CoderMJLee/Mems)
