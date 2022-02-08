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
type yser struct{
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

方法实际上也是函数，只是在声明时，在关键字`func`和方法名之间增加了一个参数
```
fuc(u user) notify(){
  fmt.Printf(u.name)

}
```
Go 语言里有两种类型的接收者
1. 值接收者:
2. 指针接收者。


