---
layout: post
title: Swift之下标
date: 2019-10-30 17:32:24.000000000 +09:00
categories: 
- Swift
---

本文首发于[个人博客](https://ityongzhen.github.io/Swift%E4%B9%8B%E4%B8%8B%E6%A0%87.html)

## 前言

Swift中对枚举、结构体、类使用下标(subscript），就可以像使用数组一样来使用了

## 使用规则

- 使用subscript可以给任意类型（枚举、结构体、类）增加下标功能，有些地方也翻译为：下标脚本
- subscript的语法类似于实例方法、计算属性，本质就是方法（函数）

例如下面的代码中，类`Point`中，的属性 x 和 y,可以用下标访问

~~~~
class Point {
    var x = 0.0, y = 0.0
    subscript(index: Int) -> Double {
        set {
            if index == 0 {
                x = newValue
            } else if index == 1 {
                y = newValue
            }
        }
        get {
            if index == 0 {
                return x
            } else if index == 1 {
                return y
            }
            return 0
        }
    }
}
~~~~

访问的时候

~~~~
var p = Point()
p[0] = 11.1
p[1] = 22.2
print(p.x) // 11.1
print(p.y) // 22.2
print(p[0]) // 11.1
print(p[1]) // 22.2

~~~~

### 注意点
-  subscript中定义的返回值类型决定了
	- get方法的返回值类型
	- set方法中newValue的类型
- subscript可以接受多个参数，并且类型任意


### subscript可以没有set方法

例如下面的代码中，只提供了get，没有set

~~~~
class Point {
    var x = 0.0, y = 0.0
    subscript(index: Int) -> Double {
        get {
            if index == 0 {
                return x
            } else if index == 1 {
                return y
            }
            return 0
        }
    }
}
~~~~

- 如果只有get方法，可以省略get


上面的代码可以写成

~~~~
class Point {
    var x = 0.0, y = 0.0
    subscript(index: Int) -> Double {
        if index == 0 {
            return x
        } else if index == 1 {
            return y
        }
        return 0
    }
}

~~~~

### 可以设置参数标签


例如下面的代码

~~~~
class Point {
    var x = 0.0, y = 0.0
    subscript(index i: Int) -> Double {
        if i == 0 {
            return x
        } else if i == 1 {
            return y
        }
        return 0
    }
}

~~~~

调用的时候

~~~~
var p = Point()
p.y = 22.2
print(p[index: 1]) // 22.2
~~~~

### 下标可以是类型方法

如下

~~~~
class Sum {
    static subscript(v1: Int, v2: Int) -> Int {
        return v1 + v2
    }
}

print(Sum[10, 20]) // 30
~~~~

## 结构体、类作为返回值对比
### 结构体

eg,如下代码,结构体Point，用了下标`subscript `只有get方法

~~~~
struct Point {
    var x = 0, y = 0
}

class PointManager {
    var point = Point()
    subscript(index: Int) -> Point {
        get { point }
    }
}
~~~~

使用的时候报错

~~~~
var pm = PointManager()
pm[0].x = 11 //Cannot assign to property: subscript is get-only
// 等价于 pm[0] = Point(x: 11, y: pm[0].y)
pm[0].y = 22//Cannot assign to property: subscript is get-only
~~~~

### 解决办法一

- 加上set方法

~~~~
struct Point {
    var x = 0, y = 0
}

class PointManager {
    var point = Point()
    subscript(index: Int) -> Point {
       set { point = newValue }
        get { point }
    }
}

//下面使用正常
var pm = PointManager()
pm[0].x = 11
// 等价于 pm[0] = Point(x: 11, y: pm[0].y)
pm[0].y = 22

~~~~

### 解决办法二

结构体改成类

~~~~
class Point {
    var x = 0, y = 0
}

class PointManager {
    var point = Point()
    subscript(index: Int) -> Point {
        get { point }
    }
}

//下面使用正常
var pm = PointManager()
pm[0].x = 11
// 等价于 pm[0] = Point(x: 11, y: pm[0].y)
pm[0].y = 22

~~~~

### 原因分析

类是引用类型的，传递的是地址
结构体是值类型，传递的是具体的值

## 接受多个参数的下标

eg,如下代码


~~~~
class Grid {
    var data = [
        [0, 1, 2],
        [3, 4, 5],
        [6, 7, 8]
    ]
    subscript(row: Int, column: Int) -> Int {
        set {
            guard row >= 0 && row < 3 && column >= 0 && column < 3 else {
                return
            }
            data[row][column] = newValue
        }
        get {
            guard row >= 0 && row < 3 && column >= 0 && column < 3 else {
                return 0
            }
            return data[row][column]
        }
    }
}

~~~~

如下使用

~~~~
var grid = Grid()
grid[0, 1] = 77
grid[1, 2] = 88
grid[2, 0] = 99
print(grid.data)
~~~~

输出为

> [[0, 77, 2], [3, 4, 88], [99, 7, 8]]






















参考资料:


[Swift官方源码](https://github.com/apple/Swift)

[从入门到精通Swift编程](https://ke.qq.com/course/392094)
