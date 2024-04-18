---
title: 工作中常用ES6语法总结
date: 2024-03-26 14:40:57
tags: JavaScript
top: 1
---

### 一、字符串扩展

1.includes(), startsWith(), endsWith()
```
let str = 'Hello World'
str.includes('Hello')   // true
str.startsWith('Hello') // true
str.endsWith('d')   // true
str.startsWith('world', 6) // true
str.endsWith('Hello', 5) // true
str.includes('Hello', 6) // false
```

2.repeat()
```
'x'.repeat(3) // "xxx"
```
##### 注意：参数不能传负数和Infinity,参数NaN为0，参数为字符串会先转换为数字

3.padStart()，padEnd()

参数两个，第一个为位数，第二个是用什么补全，省略第二个参数时默认为空格补全。这两个更多的用途是补全指定位数
```
'1'.padStart(10, '0') // "0000000001"
'12'.padStart(10, '0') // "0000000012"
'123456'.padStart(10, '0') // "0000123456"
```
4.at()

```
// 参数传入角标，返回值为角标对应的字符
'abc'.at(0) // 'a'
'吉'.at(0)  // '吉'
// 与ES5中charAt()不同之处，汉字的话ES5会返回对应Unicode编码，js内部用UTF-16
```
5.模板字符串中嵌入变量，需要将变量名写在${}之中

```
// 传统写法为
// 'User '
// + user.name
// + ' is not authorized to do '
// + action
// + '.'
ES6: `User ${user.name} is not authorized to do ${action}.`;
```

### 二、数组扩展

1.Array.from()

```
let arrayLike = {
    '0': 'a',
    '1': 'b',
    '2': 'c',
    length: 3
};

// ES5的写法
var arr1 = [].slice.call(arrayLike); // ['a', 'b', 'c']

// ES6的写法
let arr2 = Array.from(arrayLike); // ['a', 'b', 'c']
```
2....运算符

扩展运算符（spread）是三个点（...）。它好比 rest 参数的逆运算，将一个数组转为用逗号分隔的参数序列。
```
console.log(...[1, 2, 3])
// 1 2 3

console.log(1, ...[2, 3, 4], 5)
// 1 2 3 4 5
```
关于函数传参的问题，如果函数直接用...传参，传入的参数实际上是个数组，且后面不能再有参数。如果函数参数定义了一个数组，用...传入，实际上参数为数组中的值

```
function fn(...items){}
//等同于
function fn([数组内容]){}

let items = ['a', 'b', 'c']
function fn(...items){}
// 等同于
function fn('a', 'b', 'c') {}

```
运算符的应用

- 复制数组（克隆数组）

```
let arr1 = [1, 2, 3]
let arr2 = [...arr1] // [1, 2, 3]
```
- 合并数组

```
const arr1 = ['a', 'b']
const arr2 = ['c']
const arr3 = ['d', 'e']
const arr4 = [...arr1, ...arr2, ...arr3] // [ 'a', 'b', 'c', 'd', 'e' ]
```
- 与解构赋值结合

```
const [first, ...rest] = [1, 2, 3, 4, 5];
first // 1
rest  // [2, 3, 4, 5]
```
- 将字符串转换为真正的数组

```
[...'hello']
// [ "h", "e", "l", "l", "o" ]
```
3.Array.of()， 用于将一组值，转换为数组。

```
Array.of(3, 11, 8) // [3,11,8]
Array.of(3) // [3]
Array.of(3).length // 1
```
4.数组实例的fill()

三个参数，后面两个可以省略，第一个参数为替换成什么内容，第二个为替换的起始位置，第三个为替换的终止位置
```
['a', 'b', 'c'].fill(7)
// [7, 7, 7]

['a', 'b', 'c'].fill(7, 1, 2)
// ['a', 7, 'c']
```
5.数组实例的 entries()，keys() 和 values()

三个方法都是遍历数组，都可以用for...of...唯一的区别是keys()是对键名的遍历、values()是对键值的遍历，entries()是对键值对的遍历。

```
for (let index of ['a', 'b'].keys()) {
  console.log(index);
}
// 0
// 1

for (let elem of ['a', 'b'].values()) {
  console.log(elem);
}
// 'a'
// 'b'

for (let [index, elem] of ['a', 'b'].entries()) {
  console.log(index, elem);
}
// 0 "a"
// 1 "b"
```

### 三、Set和Map结构
1.Set

ES6 提供了新的数据结构 Set。它类似于数组，但是成员的值都是唯一的，没有重复的值。Set 本身是一个构造函数，用来生成Set数据结构。此结构不会添加重复的值（你想到了什么，一定想到了数组去重）

```
// 例一
const set = new Set([1, 2, 3, 4, 4]);
[...set]
// [1, 2, 3, 4]

// 例二
const items = new Set([1, 2, 3, 4, 5, 5, 5, 5]);
items.size // 5

// 例三 数组去重
[...new Set(array)]
```
2.Map

这个结构我个人感觉不如对象好用，没用过。实际上Map结构就是键值对的结构，里面设置和获取分别用set和get方法，我要设置的话每次都set一下，我觉得很不方便

3.async和await（敲黑板,async 函数是什么？一句话，它就是 Generator 函数的语法糖。）


```
注意事项：await命令只能用在async函数之中，如果用在普通函数，就会报错。
```
async函数返回一个 Promise 对象，可以使用then方法添加回调函数。当函数执行的时候，一旦遇到await就会先返回，等到异步操作完成，再接着执行函数体内后面的语句。


4.await命令

正常情况下，await命令后面是一个 Promise 对象。如果不是，会被转成一个立即resolve的 Promise 对象

```
async function f() {
  return await 123;
}

f().then(v => console.log(v))
// 123
```

