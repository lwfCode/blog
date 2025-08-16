---
title: Go语言学习之接口
date: 2025-08-16 14:05:03
tags: Go
categories: 后端
top: 31
---

#### 一、接口的概念

在Go语言中，接口 (interface) 是方法的集合, 它允许我们定义一组方法，任何类型只要实现了接口中的所有方法，就被认为是实现了该接口。
接口更重要的作用在于多态实现，它允许程序以多态的方式处理不同类型的值，接口体现了程序设计的灵活性和可扩展性，符合高内聚低耦合的设计原则。

#### 二、接口的定义

每个接口类型由任意个方法签名组成，接口的定义格式如下：
```
type 接口类型名称 interface {
    方法1(参数列表1) 返回值1
    方法2(参数列表2) 返回值2
}
type Writer interface {
    Write([]byte)(int,error)
}
```
#### 三、接口注意事项

1、接口本身不能创建实例，但是可以指向一个实现了该接口的自定义类型的变量(实例).
2、在Go语言中，一个自定义类型需要将某个接口的所有方法都实现，才能说实现了这个接口。
3、一个自定义类型实现了某个接口，才能将该自定义类型的实例赋给接口类型。
4、只要是自定义数据类型就可以实现接口，不仅仅是结构体类型，还包括类型别名、其他接口、自定义类型、变量等。
5、一个自定义类型可以实现多个接口。
6、interface接口不能包含任何变量。
7、接口类型可以作为函数的参数和返回值。
8、interface类型默认是一个指针(引用类型)，如果没有对interface初始化就使用，那么会输出nil
```
var i interface{}
fmt.Println(i == nil) //输出：true
```
9、在Go语言中，接口的实现是非侵入式的，隐式的，不需要显式声明“我实现了这个接口”。只需要一个类型提供了接口中定义的所有方法，就自动实现了该接口。

#### 四、接口的侵入式与非侵入式
###### 4.1 侵入式

侵入式接口的实现是显式声明的，必须显式的表明我要继承那个接口，必须通过特定的关键字（‌如Java中的implements）‌来声明实现关系

* 优点：通过侵入代码与你的代码结合可以更好的利用侵入代码提供给的功能。
* 缺点：框架外代码就不能使用了，不利于代码复用。依赖太多重构代码太痛苦了，把实现类与具体接口绑定起来了，强耦合。

##### 4.2 非侵入式

非侵入式接口的实现是隐式声明的，不需要通过任何关键字声明类型与接口之间的实现关系。‌只要一个类型实现了接口的所有方法，‌那么这个类型就实现了这个接口

* 优点：代码可复用，方便移植。非侵入式也体现了代码的设计原则：高内聚，低耦合。
* 缺点：无法复用框架提供的代码和功能。只能使用接口定义的方法，无法使用框架提供的方法。

#### 五、接口的应用场景

Go接口的应用场景包括多态性、‌解耦、‌扩展性、‌代码复用、API设计、‌单元测试、‌插件系统、‌依赖注入。‌

* 多态性：接口使得代码可以更加灵活地处理不同类型的数据。通过接口，可以编写更加通用的代码，而无需关心具体的数据类型。
* 解耦：‌通过接口将代码模块解耦，‌降低模块之间的耦合度。‌不同模块只需要遵循同一个接口，‌即可实现模块间的无缝整合。‌这样当一个模块的实现发生变化时，其他模块不需要做出相应的修改，只需要修改接口的实现即可。
* 扩展性：通过接口，可以轻松地为现有的类型添加新的功能，只需实现相应的接口，‌而无需修改原有的代码。这种方式使得代码更容易扩展和维护。
* 代码复用：接口提供了一种将相似行为抽象出来并进行复用的方式，从而减少了代码的重复性。这样，可以更加高效地编写和维护代码。
* API设计：‌通过定义接口，‌可以规范API的输入和输出，‌提高代码的可读性和可维护性。‌
* 单元测试：‌通过使用接口，‌可以轻松地替换被测试对象的实现，‌从而实现对被测代码的独立测试。‌
* 插件系统：‌通过定义一组接口，‌不同的插件可以实现这些接口，‌并在程序运行时动态加载和使用插件。‌
* 依赖注入：‌通过定义接口，‌可以将依赖对象的创建和管理交给外部容器，‌从而实现松耦合的代码结构。‌

##### 示例：
```
package main

import "fmt"

type User struct {
	id   int
	name string
}

type UserStore interface {
	GetUser(id int) (*User, error)
	SaveUser(user *User) error
}

type MySqlStore struct{}
type MongoStore struct{}

func (m *MySqlStore) GetUser(id int) (*User, error) {
	return &User{
		id:   1,
		name: "mysql",
	}, nil
}
func (m *MySqlStore) SaveUser(user *User) error {
	return nil
}
func (m *MongoStore) GetUser(id int) (*User, error) {
	return &User{
		id:   2,
		name: "mongodb",
	}, nil
}
func (m *MongoStore) SaveUser(user *User) error {
	return nil
}
func UserService(u UserStore) {
	userInfo, _ := u.GetUser(1)
	fmt.Println(userInfo)
}

type Payment interface {
	Pay(amount float64) (string, error)
	Refund(tradeNo string) (string, error)
	Sign(str string) error
}
type Alipay struct {
	appId string
}
type Wechat struct {
	mchId string
}

func (a *Alipay) Pay(amount float64) (string, error) {
	return fmt.Sprintf("alipay_Pay:%v, appId:%v", amount, a.appId), nil
}
func (a *Alipay) Refund(tradeNo string) (string, error) {
	str := fmt.Sprintf("alipay_Refund:%v, appId:%v", tradeNo, a.appId)
	return str, nil
}
func (a *Alipay) Sign(str string) error {
	fmt.Println("支付宝支付签名:", str)
	return nil
}
func (w *Wechat) Pay(amount float64) (string, error) {
	return fmt.Sprintf("wechat_Pay:%v, mchId:%v", amount, w.mchId), nil
}
func (w *Wechat) Refund(tradeNo string) (string, error) {
	str := fmt.Sprintf("wechat_Refund:%v, mchId:%v", tradeNo, w.mchId)
	return str, nil
}
func (w *Wechat) Sign(str string) error {
	fmt.Println("微信支付签名:", str)
	return nil
}
func PayMentService(p Payment, amount float64) {
	tradeNo, _ := p.Pay(amount)
	fmt.Println("支付单号：", tradeNo)
}
func main() {
	//依赖注入（降低模块耦合）
	//大型项目中，模块间依赖关系复杂，直接依赖具体实现会导致修改困难。通过接口定义依赖契约，可灵活替换实现（如切换数据库、缓存等）。
	//如数据库操作解耦合，实现业务代码与数据库实现完全解耦，切换数据库时无需修改业务逻辑，符合 “开闭原则”。
	UserService(&MySqlStore{})
	UserService(&MongoStore{})

	//多态处理（统一行为抽象）当不同类型需要支持相同行为时，用接口定义行为标准，实现 “同一接口，不同实现” 的多态效果。
	//如支付系统,新增支付方式（如银联）时，只需实现 Payment 接口，无需修改 ProcessPayment 等核心逻辑，扩展性极强。
	PayMentService(&Alipay{appId: "10010"}, 150000)
	PayMentService(&Wechat{mchId: "10086"}, 180000)
}
```