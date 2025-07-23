---
title: Go语言学习之常用数据类型相互转换
date: 2025-07-23 09:40:03
tags: Go
categories: 后端
top: 29
---


#### 1. 整型转字符串

使用 strconv.Itoa() 或 fmt.Sprintf()：

```
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // 方法1：strconv.Itoa（推荐，仅用于int类型）
    num1 := 123
    str1 := strconv.Itoa(num1)
    fmt.Printf("类型：%T，值：%s\n", str1, str1) // 输出：string，123

    // 方法2：fmt.Sprintf（支持各种整型，如int64、uint等）
    num2 := int64(456)
    str2 := fmt.Sprintf("%d", num2)
    fmt.Printf("类型：%T，值：%s\n", str2, str2) // 输出：string，456

    // 其他整型（如uint、uint64）转字符串
    num3 := uint(789)
    str3 := fmt.Sprintf("%d", num3)
    fmt.Printf("类型：%T，值：%s\n", str3, str3) // 输出：string，789
}
```

#### 2. 字符串转整型

使用 strconv.Atoi()（转 int）或 strconv.ParseInt()/strconv.ParseUint()（转特定长度整型）：

```
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // 方法1：strconv.Atoi（转int类型）
    str1 := "123"
    num1, err := strconv.Atoi(str1)
    if err != nil {
        fmt.Println("转换失败：", err)
    } else {
        fmt.Printf("类型：%T，值：%d\n", num1, num1) // 输出：int，123
    }

    // 方法2：strconv.ParseInt（转int64，支持指定基数和位数）
    str2 := "456"
    // 基数10，位数64（表示int64）
    num2, err := strconv.ParseInt(str2, 10, 64)
    if err != nil {
        fmt.Println("转换失败：", err)
    } else {
        fmt.Printf("类型：%T，值：%d\n", num2, num2) // 输出：int64，456
    }

    // 方法3：strconv.ParseUint（转uint64）
    str3 := "789"
    num3, err := strconv.ParseUint(str3, 10, 64)
    if err != nil {
        fmt.Println("转换失败：", err)
    } else {
        fmt.Printf("类型：%T，值：%d\n", num3, num3) // 输出：uint64，789
    }
}
```
### 注意：字符串转整型时必须处理错误（如字符串非数字时会报错）。

#### 3、浮点型 ↔ 字符串

```
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // 浮点型转字符串
    floatNum := 3.14159
    str1 := fmt.Sprintf("%f", floatNum)       // 保留6位小数："3.141590"
    str2 := fmt.Sprintf("%.2f", floatNum)     // 保留2位小数："3.14"
    str3 := strconv.FormatFloat(floatNum, 'f', 2, 64) // 等价于%.2f："3.14"

    // 字符串转浮点型
    str4 := "3.14"
    floatNum2, err := strconv.ParseFloat(str4, 64)
    if err == nil {
        fmt.Printf("类型：%T，值：%v\n", floatNum2, floatNum2) // 输出：float64，3.14
    }
}
```

#### 4、布尔值 ↔ 字符串

```
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // 布尔值转字符串
    b := true
    str1 := fmt.Sprintf("%t", b) // "true"
    str2 := strconv.FormatBool(b) // "true"

    // 字符串转布尔值（仅"true"或"false"可转，不区分大小写）
    str3 := "false"
    b2, err := strconv.ParseBool(str3)
    if err == nil {
        fmt.Printf("类型：%T，值：%v\n", b2, b2) // 输出：bool，false
    }
}
```

#### 5、不同长度整型之间的转换（如 int ↔ int64）

Go 语言中不同长度的整型（如 int、int64、uint 等）需要显式转换：

```
package main

import "fmt"

func main() {
    var a int = 100
    var b int64 = int64(a) // int 转 int64
    var c uint = uint(b)   // int64 转 uint（注意：可能溢出，需谨慎）

    fmt.Printf("a: %T = %d\n", a, a) // int = 100
    fmt.Printf("b: %T = %d\n", b, b) // int64 = 100
    fmt.Printf("c: %T = %d\n", c, c) // uint = 100
}
```
### 注意：转换时需注意范围，例如 int64 转 int 可能因超出范围导致溢出。

#### 6、字符串 ↔ 字节切片（[] byte）

```
package main

import "fmt"

func main() {
    // 字符串转字节切片
    str := "hello"
    b := []byte(str)
    fmt.Printf("类型：%T，值：%v\n", b, b) // []uint8，[104 101 108 108 111]

    // 字节切片转字符串
    b2 := []byte{104, 101, 108, 108, 111}
    str2 := string(b2)
    fmt.Printf("类型：%T，值：%s\n", str2, str2) // string，hello
}
```

### 总结

> 整型 ↔ 字符串：用 strconv.Itoa/Atoi（int 专用）或 fmt.Sprintf/strconv.ParseXXX（支持多类型）。
> 浮点型 ↔ 字符串：用 fmt.Sprintf 或 strconv.FormatFloat/ParseFloat。
> 布尔值 ↔ 字符串：用 strconv.FormatBool/ParseBool。
> 不同长度整型：直接显式转换（如 int64(a)），注意范围溢出。
> 字符串 ↔ 字节切片：直接通过类型转换符 []byte(str) 或 string(b)。