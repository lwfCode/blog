---
title: Go语言学习之匿名函数与闭包
date: 2025-07-25 09:40:03
tags: Go
categories: 后端
top: 30
---

#### 一、匿名函数

匿名函数是没有名称的函数，可以直接定义并使用，也可以赋值给变量或作为参数传递

##### 定义格式

```
func(参数)(返回值){
    函数体
}
```

匿名函数因为没有函数名，所以没办法像普通函数那样调用，所以匿名函数需要保存到某个变量或者作为立即执行函数

```
package main

import "fmt"

func main() {
    // 定义匿名函数并立即执行（自执行函数）
    func(msg string) {
        fmt.Println("匿名函数：", msg)
    }("Hello, 匿名函数") // 输出：匿名函数： Hello, 匿名函数

    // 将匿名函数赋值给变量
    add := func(a, b int) int {
        return a + b
    }
    fmt.Println("3 + 5 =", add(3, 5)) // 输出：3 + 5 = 8
}
```
##### 特点
> 没有函数名，通过 func 关键字直接定义。
> 可直接调用（自执行），也可作为变量存储或作为参数传递给其他函数。
> 语法简洁，适合临时使用的简单逻辑。

#### 二、闭包

1、闭包指的是一个函数和与其相关的引用环境组合而成的实体。简单来说，闭包=函数+引用环境。
2、闭包是可以捕获外部作用域变量的匿名函数，即使外部作用域已经结束，闭包仍能访问和修改这些变量

基本示例：

```
func adder() func(int) int {
	var x int
	return func(y int) int {
		x += y
		return x
	}
}
func main() {
	var f = adder()
	fmt.Println(f(10)) //10
	fmt.Println(f(20)) //30
	fmt.Println(f(30)) //60

	f1 := adder()
	fmt.Println(f1(40)) //40
	fmt.Println(f1(50)) //90
}
```
变量 f 是一个函数并且它引用了其外部作用域中的x变量，此时 f 就是一个闭包。 在 f 的生命周期内，变量 x 也一直有效。

进阶示例：

```
func makeSuffixFunc(suffix string) func(string) string {
	return func(name string) string {
		if !strings.HasSuffix(name, suffix) {
			return name + suffix
		}
		return name
	}
}

func main() {
	jpgFunc := makeSuffixFunc(".jpg")
	txtFunc := makeSuffixFunc(".txt")
	fmt.Println(jpgFunc("test")) //test.jpg
	fmt.Println(txtFunc("test")) //test.txt
}
```

#### 闭包注意事项

1、变量引用而非复制：闭包捕获的是变量的引用，而非值。如果外部变量被修改，闭包中访问的值也会变化

```
func main() {
    x := 10
    f := func() { fmt.Println(x) }

    x = 20
    f() // 输出：20（闭包访问的是最新值）
}
```

2、循环中的闭包陷阱：在循环中定义闭包时，若直接引用循环变量，可能因变量共享导致意外结果

```
func main() {
    funcs := []func(){}
    for i := 0; i < 3; i++ {
        // 错误：所有闭包共享同一个i的引用
        funcs = append(funcs, func() { fmt.Println(i) })
    }
    for _, f := range funcs {
        f() // 输出：3 3 3（而非0 1 2）
    }
}

解决方法：每次循环创建局部变量副本

for i := 0; i < 3; i++ {
    iCopy := i // 复制当前i的值
    funcs = append(funcs, func() { fmt.Println(iCopy) })
}
// 输出：0 1 2
```

#### 总结：

> 匿名函数：无名称的函数，适合临时使用或作为参数传递。
> 闭包：能捕获外部变量的匿名函数，可延长变量生命周期、保持状态，常用于实现计数器、回调、封装等场景。
> 核心区别：匿名函数是 “没有名字的函数”，而闭包是 “捕获了外部变量的匿名函数”（闭包是匿名函数的一种特殊形式）。