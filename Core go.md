# GMP


----
# Go Conext
> Context的传递的本质是链表，效率并不高(复杂度o(n))，

## Why context 

简单的一个指令go function就能启动一个goroutine。但是，Go语言并没有提供终止goroutine的接口，也就是说，我们不能从外部去停止一个goroutine，只能由goroutine内部退出(main函数终止除外

当最上层的 Goroutine 因为某些原因执行失败时，下层的 Goroutine 由于没有接收到这个信号所以会继续工作；因此，如何控制goroutine，在下层及时停掉无用的工作以减少额外资源的消耗：

有很多情况下需要主动关闭goroutine，主动关闭goroutine除了实现特定功能外，还能提升程序性能。goroutine由于某种原因阻塞，不能继续运行，此时程序应该干预，将goroutine结束，而不是让他一直阻塞，如果此类goroutine很多，会耗费更多的资源。因此，有效的管理goroutine是十分有必要的

1. 全局变量
2. Channnel： 层级gorutine
3. WaitGroup： WaitGroup只会等待子Goroutine结束，并不能主动通知子Goroutine退出
4. context：

**作用**
> 用一句话总结context：context 用来解决 goroutine 之间<退出通知>、<元数据传递>的功能。

1.退出通知机制（流程控制），通知可以传递到整个goroutine调用树上的每一个goroutine
	- 流程控制 :Context的Done方法主要就是用来做流程控制，用来说明当前的处理流程是否需要被取消 
	- Done: 返回一个通道，如果通道关闭则代表该Context已经被取消；如果返回的为nil，则代表该Context是一个永远不会被取消的Context。

2.传递数据，数据可以传递给整个gortouine调用树上的每一个goroutine
	- 传递上下文信息 ： - 在API请求中，将用户的鉴权信息放在context中， 字节的logID，trace span等追踪信息 
  
 
## context实现原理

利用了channel struct{}的特性，使用select获取channel数据。
- 一旦关闭这个channel则会收到数据退出goroutine中的逻辑。
- context也是支持嵌套使用，结构就如下图显示利用的是一个map类型来存储子context。关闭一个节点就会循环关闭这个节点下面的所有子节点，就实现了优雅的退出goroutine的功能。
 
go context是由多个调用链构成的树形结构,而当context树的某个中间节点被中断，后面的子节点也会“断开,如果某个value希望在整个链路总存储下来context会为这个value创建新的节点
context 中的上下文数据并不是全局的，它只能查询该节点及父节点们的数据，不能查询兄弟节点的数据

特点
- 是不可变的(immutable)树节点
- Cancel 一个节点，会连带 Cancel 其所有子节点 （从上到下）
- Context values 是一个节点
- Value 查找是回溯树的方式 （从下到上）


## 该如何使用context 

1. 传递共享数据
2. 

## Struct
```
// Context接口,下面定义的四个方法都是幂等的
type Context interface {

 // context 是否会被取消以及自动取消时间（即 deadline）

   //返回 context.Context被取消的时间，也就是完成工作的截止日期；如果没有设置截止时间，ok的值返回的是false,

   Deadline() (deadline time.Time, ok bool)

   // See https://blog.golang.org/pipelines for more examples of how to use

   // 当 context 被取消或者到了 deadline，返回一个被关闭的 channel

   //返回一个 Channel，这个 Channel 会在当前工作完成或者上下文被取消后关闭，多次调用 

   //Done方法会返回同一个 Channel；

   Done() <-chan struct{}

   //返回 context.Context结束的原因，它只会在 Done方法对应的 Channel 关闭时返回非空的值；如果 context.Context 被取消，会返回 Canceled 错误；

   //如果 context.Context 超时，会返回 DeadlineExceeded 错误；

   Err() error

   // 获取 key 对应的 value

  // 从 context.Context中获取键对应的值，对于同一个上下文来说，多次调用 Value并传入相同的 

  //Key会返回相同的结果，该方法可以用来传递请求特定的数据；

   Value(key interface{}) interface{}

}
```

Go内置已经帮我们实现了2个
```
var (

   background = new(emptyCtx)

   todo       = new(emptyCtx)

)

func Background() Context {

   return background

}

func TODO() Context {

   return todo

}
```
* 一个是Background，在主服务里面通常作为匿名参数生成可控的context变量，主要用于main函数、初始化以及测试代码中，作为Context这个树结构的最顶层的Context，也就是根Context
* 一个是TODO,它目前还不知道具体的使用场景，可能会用于程序兼容处理，当空桩，如果我们不知道该使用什么Context的时候，可以使用这个



> Context 为什么是并发安全的？

Context 本身的实现是不可变的（immutable），既然不可变，那当然是线程安全的。并且通过 Context.Done() 返回的通道可以协调 goroutine 的行为

----
# 内存管理

- 栈（Stack）：每个函数都有自己独立的栈空间，函数的调用参数、返回值以及局部变量大都被分配到该函数的栈空间中
- 堆（Heap）：堆空间没有特定的结构，也没有固定的大小，可以动态进行分配和调整，所以内存占用较大的局部变量会放在堆空间上，在编译时不知道该分配多少大小的变量，在运行时也会分配到堆上

**内存逃逸**
> 本该分配到函数栈空间的变量，被分配到了堆空间，称为内存逃逸；过多的内存逃逸会导致 GC 压力变大，堆空间内存碎片化。

只要局部变量不能证明在函数结束后不能被引用，那么就分配到堆上(如果局部变量被其他函数所捕获，那么就被分配到堆上)

两个不变性
- 指向栈对象的指针不能存储在堆中
- 指向栈对象的指针不能超过该栈对象的存活期（即指针不能在栈对象被销毁后依旧存活）



分析工具

```
go build -gcflags '-m -l' xxxx.go
```
## 逃逸场景

1. 函数返回局部变量指针
  - 在函数之外引用该局部变量的地址,编译时能发现，指针不能在栈对象被销毁后依旧存活
2.  动态反射 interface{} 变量
  - 通过 reflect.TypeOf(arg).Kind() 获取接口对象的底层数据类型，创建具体类型对象时，会发生内存逃逸。由于 interface{} 的变量，编译时无法确定变量类型以及申请空间大小，所以不能在栈空间上申请内存，需要在 runtime 时动态申请
3. 申请栈空间过大
  - 栈空间大小是有限的，如果编译时发现局部变量申请的空间过大，则会发生内存逃逸，在堆空间上给大变量分配内存
4. 切片变量自身和元素的逃逸
  - 未指定 slice 的 length 和 cap 时，slice 自身未发生逃逸，slice 的元素发生了逃逸。因为 slice 会动态扩容，编译器不知道容量大小，无法提前在栈空间分配内存
5. 发送指针或带有指针的值到 channel 中
---

