---
layout: post
title: go DeepEqual
subtitle: go中的对象比较
date: 2020-04-08
categories: Go
cover: 'assets/img/reflect.jpg'
tags: go reflect DeepEqual 
---

当我在go中比较2个对象是否相等时，我用==操作时，报了一个错误
```
a := []int{1,2}
b := []int{1,2}
fmt.Println(a == b)
```
输出
```
invalid operation: a == b (slice can only be compared to nil)
```
我试着用is去比较时，又被告知语法错误 syntax error: unexpected is, expecting comma or )。
直到我遇到的DeepEqual,DeepEqual可以对复杂结构进行比较，DeepEqual在reflect包中。<br>
> func DeepEqual(x, y interface{}) bool<br>
DeepEqual reports whether x and y are "deeply equal," defined as follows. Two values of identical type are deeply equal if one of the following cases applies. Values of distinct types are never deeply equal.

比如，我想对2个结构体实例进行比较时
```
type People struct{
    Name string
    age int
}

func f3(){
    p1 := People{Name: "aa",age: 10}
    p2 := People{age: 10,Name: "aa"}
    fmt.Print(reflect.DeepEqual(p1,p2))
}

func main(){
    f3()
}
```
输出 true <br/>
**当结构体对应的字段(公有或私有)是deeply equal时，结构体是相等的。**

```
a := make([]string,5,5)
b := make([]string,5,6)
fmt.Println(reflect.DeepEqual(a,b))
```
输出 false
```
a := []int{0,0}
b := make([]int,2)
fmt.Println(reflect.DeepEqual(a,b))
```
输出 true <br>
**当切片长度相等并且对应元素是deeply equal时，切片是相等的。**



