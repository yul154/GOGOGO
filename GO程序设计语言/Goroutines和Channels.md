# Goroutines

每一个并发的执行单元叫作一个goroutine

当一个程序启动时，其主函数即在一个单独的goroutine中运行，我们叫它main goroutine。


除了从主函数退出或者直接终止程序之外，没有其它的编程方法能够让一个goroutine来打断另一个的执行



# Channels
> goroutine间的通信机制

一个channels是一个通信机制，它可以让一个goroutine通过它给另一个goroutine发送值信息
* 每个channel都有一个特殊的类型
* 两个相同类型的channel可以使用==运算符比较。如果两个channel引用的是相通的对象，那么比较的结果为真


一个channel有发送和接受两个主要操作
* 发送和接收两个操作都是用`<‐`运算符
* 一个发送语句将一个值从一个goroutine通过channel发送到另一个执行接收操作的goroutine

```
ch <‐ x // a send statement
x = <‐ch // a receive expression in an assignment statement
<‐ch // a receive statement; result is discarded
close(ch)
```

以最简单方式调用make函数创建的时一个无缓存的channel
```
ch = make(chan int) // unbuffered channel
ch = make(chan int, 0) // unbuffered channel
ch = make(chan int, 3) // buffered channel with capacity 3
```

## 不带缓存的Channe

一个基于无缓存Channels的发送操作将导致发送者goroutine阻塞,直到另一个goroutine在相同的Channels上执行接收操作
* 如果接收操作先发生，那么接收者goroutine也将阻塞，直到有另一个goroutine在相同的Channels上执行发送操作。


基于channels发送消息有两个重要方面
* 每个消息都有一个值
* 通讯的事实和发生的时刻也同样重要。当我们更希望强调通讯发生的时刻时，我们将它称为消息事件


##  串联的Channels（Pipeline）
> Channels也可以用于将多个goroutine链接在一起,一个Channels的输出作为下一个Channels的输入。这种串联的Channels就是所谓的管道 pipeline

通过Channels只发送有限的数列该如何处理呢

1. 当一个channel被关闭后，再向该channel发送数据将导致panic异常。
2. 当一个被关闭的channel中已经发送的数据都被成功接收后，后续的接收操作将不再阻塞，它们会立即返回一个零值
3. 是接收操作有一个变体形式：它多接收一个结果，多接收的第二个结果是一个布尔值ok，ture表示成功从channels接收到值，false表示channels已经
被关闭并且里面没有值可接收

## 单方向的Channel

当一个channel作为一个函数参数是，它一般总是被专门用于只发送或者只接收
* 类型`chan<‐ int`表示一个只发送int的channel，只能发送不能接收。相反，
* 类型`<‐chan int`表示一个只接收int的channel
* 只有在发送者所在的goroutine才会调
* 用close函数，因此对一个只接收的channel调用close将是一个编译错误


## 带缓存的Channels
```
ch = make(chan string, 3）
```
<img width="250" alt="截屏2022-02-18 上午11 07 18" src="https://user-images.githubusercontent.com/27160394/154610018-c6049285-4a9e-4711-8561-3a85f2dbb87e.png">

* 向缓存Channel的发送操作就是向内部缓存队列的尾部插入元素
* 接收操作则是从队列的头部删除元素
* 如果内部缓存队列是满的，那么发送操作将阻塞直到因另一个goroutine执行接收操作而释放了新的队列空间
* 如果channel是空的，接收操作将阻塞直到有另一个goroutine执行发送操作而向队列插入元素

```
fmt.Println(cap(ch))
fmt.Println(len(ch))
```
* `channel`内部缓存的容量，可以用内置的`cap`函数获取
* `len`将返回channel内部缓存队列中有效元素的个数

使用了无缓存的channel，那么两个慢的goroutines将会因为没有人接收而被永远卡住。这种情况，称为goroutines泄漏
* 泄漏的goroutines并不会被自动回收，因此确保每个不再需要的goroutine能正常退出是重要的


`goroutines`和`channel`的选择
* 无缓存channel更强地保证了每个发送操作与相应的同步接收操作
* `channel`的缓存队列的容量设置为1。只要每个厨师的平均工作效率相近，那么其中大部分的传输工作将是迅速的，个体之间细小的效率差异将在交接过程中弥补
* 如果厨师之间有更大的额外空间——也是就更大容量的缓存队列——将可以在不停止生产线的前提下消除更大的效率波动
* 如果生产线的前期阶段一直快于后续阶段，那么它们之间的缓存在大部分时间都将是满的
* 如果后续阶段比前期阶段更快，那么它们之间的缓存在大部分时间都将是空的,额外的缓存并没有带来任何好处。
* ，如果第二阶段是需要时间长，无法满足前一个阶段的需求。要解决这个问题，创建另一个独立的goroutine


## 并发的循环

子问题都是完全彼此独立的问题被叫做易并行问题


## 基于select的多路复用

## 并发的退出

需要通知`goroutine`停止它正在干的事情
1. 创建一个退出的`channel`，这个`channel`不会向其中发送任何值，但其所在的闭包内要写明程序需要退出
2. 同时还定义了一个工具函数，`cancelled`，这个函数在被调用的时候会轮询退出状态

```
var done = make(chan struct{})
func cancelled() bool {
  select {
    case <‐done:
      return true
    default:
      return false
  }
}
```


