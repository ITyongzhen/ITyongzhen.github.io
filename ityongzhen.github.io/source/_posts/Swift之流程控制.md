---
layout: post
title: Swift之流程控制
date: 2019-05-30 17:32:24.000000000 +09:00
categories: 
- Swift
---



本文首发于[我的个人博客](https://ityongzhen.github.io/categories/Swift/)

## if-else 

- if后面的条件可以省略小括号
- 条件后面的大括号不可以省略

下面的代码是可以的

~~~~
let score = 70
if score >= 80 {
    print("优秀")
}else if score >= 60{
    print("及格")
}else{
    print("不及格")
}

~~~~

- if后面的条件只能是bool类型

例如下面是不可以的

~~~~
if score {
    print("aa")
}
~~~~


## while
先看如下代码

~~~~
var num = 5
while num > 0 {
    print("num is \(num)")
    num -= 1
}// 会打印五次


var num = 5
repeat {
    print("num is \(num)")
     num -= 1
}while num > 0// 会打印五次

~~~~

- repeat-while 相当于C语言中的 do-while
- 上面代码中没有用num--，是因为从Swift3开始，去掉了自增(++)、自减(--)运算符


## for
### 闭区间运算符: a...b，代表着: a <= 取值 <= b
例如

~~~~
let persons = ["zhangsan","lisi","wanger","mazi"]
for i in 0...3 {
    print(persons[i])
}
//结果为
//zhangsan
//lisi
//wanger
//mazi

~~~~

- 我们也可以用range来表示区间，例如

~~~~
let persons = ["zhangsan","lisi","wanger","mazi"]
let range = 0...3
for i in range {
    print(persons[i])
}
//结果为
//zhangsan
//lisi
//wanger
//mazi

~~~~

- 我们也可以用变量来表示区间，例如

~~~~
let persons = ["zhangsan","lisi","wanger","mazi"]
let before = 0
var end = 3
for i in before...end {
    print(persons[i])
}
//结果为
//zhangsan
//lisi
//wanger
//mazi

~~~~

- 我们也可以用变量和数值共同使用来表示区间，例如

~~~~
let persons = ["zhangsan","lisi","wanger","mazi"]
for i in before...3 {
    print(persons[i])
}
//结果为
//zhangsan
//lisi
//wanger
//mazi

~~~~

### 半开区间运算符：a..<b 表示 a <= 取值 < b

例如

~~~~
let persons = ["zhangsan","lisi","wanger","mazi"]
for i in 1..<3 {
    print(persons[i])
}
//结果为
//lisi
//wanger

~~~~

3. 区间运算符用在数组上
例如

~~~~
let nums = [1,2,3,4]
for num in nums[0...3] {
    print(num)
}
//结果为
//1
//2
//3
//4

~~~~

- 单侧区间

~~~~
let nums = [1,2,3,4]
for num in nums[0...] {
    print(num)
}
//结果为
//1
//2
//3
//4

~~~~

或者

~~~~
let nums = [1,2,3,4]
for num in nums[..<4] {
    print(num)
}
//结果为
//1
//2
//3
//4

~~~~

## 区间类型
如下三种

- let range1: ClosedRange<Int> = 1...3
- let range2: Range<Int> = 1..<3
- let range3: PartialRangeThrough<Int> = ...5

字符、字符串也可以使用区间运算符，但默认不能使用在for-in中
例如

~~~~
这样写是可以的
let strRange = "a"..."f"
strRange.contains("d") // true
strRange.contains("l") // false

但是下面是会报错的
for i in strRange {
    print(i)
}
~~~~

- \0 到 "~" 包括了所有的要用到的ASCII字符
例如我们要判断一个字符是否是ASCII字符

~~~~
let characterRange: ClosedRange<Character> = "\0"..."~"
//想判断s是否是ASCII字符可以
characterRange.contains("s") //返回true
~~~~

### 带间隔的区间值

用到了 stride 
看代码

~~~~
let all = 100
let interval = 20
// res的取值为从 10 开始 每次间隔 20，直到100结束，
for res in stride(from: 10, to: all, by: interval) {
    print(res)
}// 结果为
//10
//30
//50
//70
//90

~~~~

## switch
 
- case、default 后面不能写大括号{}
- 默认可以不写break，并不会贯穿到后面的条件

例如

~~~~
var res = 1
switch res {
case 0:
    print("res = 0")
case 1:
    print("res = 1")
case 2:
    print("res = 2")
default:
     print("other res")
}
// 输出为 res = 1
~~~~

**关键字 fallthrough**

如果我们想让其贯穿下去，就是用 fallthrough 这个关键字
例如

~~~~
var res = 1
switch res {
case 0:
    print("res = 0")
case 1:
    print("res = 1")
    fallthrough
case 2:
    print("res = 2")
default:
     print("other res")
}
// 输出为
// res = 1
// res = 2
~~~~

### switch中支持 字符串，字符类型
例如

~~~~
字符串
let string = "aaa"
switch string {
case "aaa":
    print("string is aaa")
case "bbb":
    print("string is bbb")
default:
    break
} // string is aaa

字符类型
let character: Character = "a"
switch character {
case "a","A":
     print("string is a or A")
default:
    print("string is not a or A")
} //string is a or A

~~~~

### 区间、元组匹配
- 可以用下划线_ 忽略某个值
- 可以对区间，和元组进行匹配

~~~~
let count = 8
switch count {
case 0:
    print("0")
case 1..<5:
    print("1到4")
case 5..<10:
    print("5到10")
default:
    break
}
//5到10
~~~~

和

~~~~
let point = (1,0)
switch point {
case (0, 0):
    print("原点")
case (_, 0):
    print("x轴")
case (0, _):
    print("y轴")
case (-2...2, -2...2):
    print("区间")

default:
     print("other")
}
//x轴
~~~~

### 值绑定

~~~~
let point2 = (1,0)
switch point2 {
case (0, 0):
    print("原点")
case (let x, 0):
    print("x轴 x是 \(x)")
case (0, let y):
    print("y轴 y是 \(y)")
case let (x, y):
    print("somewhere else at (\(x),\(y))")
    
default:
    print("other")
}
// x轴 x是 1

~~~~


## where


~~~~
var numbers = [1,2,3,4,5,]
var sum = 0
for num in numbers where num > 2 {
    sum += num
}
print(sum) //12

~~~~


## 标签语句

标签语句用于执行的时候，跳转到标签的位置

例如

~~~~
outer: for i in 1...4{
    for k in 1...4 {
        if k == 2 {
            continue
        }
        if i == 3 {
            break outer
        }
         print("i == \(i), k == \(k)")
    }
}
输出为 
i == 1, k == 1
i == 1, k == 3
i == 1, k == 4
i == 2, k == 1
i == 2, k == 3
i == 2, k == 4

~~~~

如果加了标签

~~~~
outer: for i in 1...4{
    for k in 1...4 {
        if k == 2 {
            continue outer
        }
        if i == 3 {
            break outer
        }
         print("i == \(i), k == \(k)")
    }
   
}
输出为 
i == 1, k == 1
i == 2, k == 1

~~~~




参考资料:

[从入门到精通Swift编程](https://ke.qq.com/course/392094)

[Swift官方源码](https://github.com/apple/Swift)

