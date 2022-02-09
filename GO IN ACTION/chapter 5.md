* 声明新的用户定义的类型
* 使用方法，为类型增加新的行为
* 了解何时使用指针，何时使用值
* 通过接口实现多态
* 通过组合来扩展或改变类型
* 公开或者未公开的标识符
-----
go是静态语言，编译时需要知道值的类型
* 需要分配多少内存给这个值
* 这段内存表示什么

----
# 用户定义的类型

Go 语言里声明用户定义的类型有两种方法


1. `struct`可以定义一个结构类型

```
type user struct{
    name string
    email string
    ext   int
    privileged bool
  
}

var bill user

leo := user{

    name: "leo",
    email: "leo@bytedance.com",
    ext: 123,
    priviledged: true,
}

lisa := user{"Lisa", "lisa@email.com", 123, true} //
```

2. 基于一个已有的类型，将其作为新类型的类型说明。
```
type Duration int64
```
----
# 方法
> 方法提供了一种给用户定义的类型增加行为的方式。



方法实际上也是函数，只是在声明时，在关键字`func`和方法名之间增加了一个参数
* 将函数与接收者的类型绑在一起
* 把函数名以大写字母开头，该函数的作用域就大了，可以被其他包调用,小写只能在本包使用
```
fuc Add(a,b,int) int { //function - 函数

    return a+b
}

fuc(u user) notify() string{ //method - 方法

  return u.name

}

kelly :=user{}
fmt.print(kelly.notify())

```


Go 语言里有两种类型的接收者
1. 值接收者: 使用值接收者声明方法，调用时会使用这个值的一个副本来执行
2. 指针接收者: 而指针接受者使用实际值来调用方法
* 如果是要创建一个新值，该类型的方法就使用值接收者。如果是要修改当前值，就使用指针接收者

```
kelly :=user{}
fmt.print(kelly.notify())

lisa := &user{"Lisa", "lisa@email.com"}
lisa.notify() //指针引用
(*lisa).notify() // 值引用
```
----

# 类型的本质

## 内置类型 

当对这些值进行增加或者删除的时候，会创建一个新值

## 引用类型 

Go 语言里的引用类型有：切片、映射、通道、接口和函数类型

创建的变量被称作标头（header）值
* 每个引用类型创建的标头值是包含一个指向底层数据结构的指针
* 每个引用类型还包含一组独特的字段，用于管理底层数据结

结构类型
* 一组数据值
```
type Time structure{

    sec int64
    nsec int32
    loc *Location

}
```

----

# 接口

多态：如果一个类型实现了某个接口，所有使用这个接口的地方，都可以支持这种类型的值

## 实现　
> 接口是用来定义行为的类型

接口值是一个两个字长度的数据结构
1. 包含一个指向内部表的指针。
    * 这个内部表叫作 iTable，包含了所存储的值的类型信息。
    * iTable 包含了已存储的值的类型信息以及与这个值相关联的一组方法。
2. 一个指向所存储值的指针。

![截屏2022-02-09 下午5 44 57](https://user-images.githubusercontent.com/27160394/153170361-397619e5-4f75-4761-b3c0-849b02cb147c.png)


## 方法集
> 定义了接口的接受规则

方法集定义了一组关联到给定类型的值或者指针的方法
![截屏2022-02-09 下午5 58 32](https://user-images.githubusercontent.com/27160394/153172851-c7d4ef99-3fdc-487a-8fc6-25677497c46a.png)

![截屏2022-02-09 下午5 58 58](https://user-images.githubusercontent.com/27160394/153172928-2be386f9-ab10-4f13-b661-93a857028d2c.png)

## 多态

```
package main 
import ("fmt")

type notifier interface( notify())

type user struct{
    
    name string
    email string
 }

type admin struct{

    name string
    email string

}

func (u *user) notify(){

    fmt.print("sending email to"+ u.name+u.email )
}

func (u *admin) notify(){

    fmt.print("sending email to"+ u.name+u.email )
}

```
---
# 嵌入类型 - type embedding

嵌入类型是将已有的类型直接声明在新的结构类型里
* 被嵌入的类型被称为新的外部类型的内部类型

```
package main 
import ("fmt")

type notifier interface( notify())

type user struct{
    
    name string
    email string
 }

type admin struct{

    user // user 是外部类型 admin 的内部类型
    level string

}
```
* 与内部类型相关的标识符会提升到外部类型上。这些被提升的标识符就像直接声明在外部类型里的标识符一样，也是外部类型的一部分
* 如果外部类型实现了 notify 方法，内部类型的实现就不会被提升

# 公开或未公开的标识符（modifier）

* 使用与代码所在文件夹一样的名字作为包名
* 标识符的名字以小写字母开头时，这个标识符就是未公开的，即包外的代码不可见。
* 标识符以大写字母开头，这个标识符就是公开的，即被包外的代码可见。
* 将工厂函数命名为 New 是 Go 语言的一个习惯。



* 由于内部类型 user 是未公开的，这段代码无法直接通过结构字面量的方式初
始化该内部类型。
* 不过，即便内部类型是未公开的，内部类型里声明的字段依旧是公开的。
* 既然内部类型的标识符提升到了外部类型，这些公开的字段也可以通过外部类型的字段的值来访问
---
# QUESTION
