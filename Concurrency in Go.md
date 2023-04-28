# Basic Concept

**同步和异步**
- 同步方法调用开始之后，被调用者会持续执行任务，在执行完成之后进行返回。而在被调用者返回之前，调用者会被阻塞，直到调用结束后，才执行之后的代码。
- 异步调用则相反，调用者会在发起调用之后继续执行下面的代码，而不阻塞地等待被调用者的返回

**并行与并发**
- 并行指的是在任一时刻，两个任务物理上都在运行之中。并行依赖于物理上的多核心实现。
- 并发指的是在一段时间之内，宏观上看，两个任务都被调度。并发的实现通常是时间片算法，让两个任务看上去像是“同时运行”的

**进程 VS 线程 VS 协程**

- 进程是计算机进行资源分配和调度的基本单位,一个进程的主要组成部分包括程序指令集、数据、程序控制块（PCB）
- 线程是操作系统进行任务调度和处理器分配的基本单位, 一个进程可以拥有多个线程
- 协程 (Coroutines) 是比线程更加轻量化的任务，具有内核不可见性的特征，是由开发者编写的程序来进行管理的

协程与线程的映射关系

![image](https://user-images.githubusercontent.com/27160394/235094526-d200cf9a-3ad5-4d13-a62f-0f320ca3ddbb.png)


![image](https://user-images.githubusercontent.com/27160394/235094144-d257319f-cd1a-448f-9dc4-e68bc879e4a0.png)

![image](https://user-images.githubusercontent.com/27160394/235094441-9c8bc434-e458-4ac0-aedb-f79ed3d21f6c.png)


----
# Go 中的并发
> 不要以共享内存的方式来通信；相反，要通过通信的方式来共享内存。

CSP(Communicating Sequential Processes): DO NOT COMMUNICATE BY SHARING MEMORY; INSTEAD, SHARE MEMORY BY COMMUNICATING.


## GMP
> Go 采用一种类似 M : N 协程映射关系的模型

> https://strikefreedom.top/archives/high-performance-implementation-of-goroutine-pool#toc-head-5

**G (goroutine)**
- G 表示 Goroutine，它是一个待执行的任务。
- Goroutine 的调度都由go进行运行时管理。
- Goroutine 执行异步操作时会进入休眠状态，待操作完成后再恢复，无需占用系统线程。
- Goroutine 新建或恢复时会添加到运行队列，等待M取出并运行

**M (machine)**
- 表示操作系统的线程，它由操作系统的调度器调度和管理。
- 当没有足够的 M 来关联 P 并运行其中的可运行的 G 时，会创建新的 M
- Go 默认限定M的最多10000
- 可以通过`runtime/debug`包中的`SetMaxThreads`函数来设置最大数量
- 有⼀个M阻塞，会创建⼀个新的M. 如果有M空闲，那么就会回收或者睡眠


**P (processor)**
- P 指的是能够活跃使用的计算资源，可以通过环境变量GOMAXPROC修改
- 在确定了 P 的最大数量后，运行时系统会根据这个数量创建 P

![image](https://user-images.githubusercontent.com/27160394/235097853-1ed3e190-76dc-4c3b-b540-914b66b53648.png)

**实际关系**
- `goroutine`（称为 G）也不例外，但是 G 并不直接绑定 OS 线程运行，而是由 Goroutine Scheduler 中的 P - Logical Processor （逻辑处理器）来作为两者的中介
- P 可以看作是一个抽象的资源或者一个上下文，一个 P 绑定一个 OS 线程
- 在 golang 的实现里把 OS 线程抽象成一个数据结构：M
- G 实际上是由 M 通过 P 来进行调度运行的
- 在 G 的层面来看，P 提供了 G 运行所需的一切资源和环境，因此在 G 看来 P 就是运行它的 “CPU”
- P 持有一个由可运行的`Goroutine`组成的环形的运行队列 runq，还反向持有一个线程。调度器在调度时会从 P 的队列中选择队列头的 Goroutine 放到线程 M 上执行


### 可运行队列 `runq`

可运行队列的结构：三级队列
![image](https://user-images.githubusercontent.com/27160394/235099629-4c36579a-33d3-4268-85df-c08fe1070a5e.png)

`runnext`实际上只能指向一个 goroutine，是一个特殊的队列
* 如果`runnext`为空，那么 goroutine 就会顺利地放入`runnext`，之后以最高优先级得到消费
* 如果`runnext`不为空，则要替换掉`runnext`上的 old goroutine。老的g被替换到哪里呢？
    1. local queue 是一个大小为 256 的数组，实际上用 head 和 tail 指针把它当成一个环形数组在使用。如果 local queue 不满，则将`runnext`放入`local queue`
    2. 否则，P的本地队列上的`goroutine`太多了，说明当前 P 的任务太重了，需要减负，因此需要得到其他 P 协助。从而，将`runnext`以及当前 P 的一半`goroutine`一起加入到`global queue`里去。


在寻找goroutine时，根据可执行队列来找:`runnext` -> `local `-> `global`

**Goroutine 状态**
运行期间会在这三种状态来回切换：
- 等待中：Goroutine 正在等待某些条件满足，例如：系统调用结束等，包括 _Gwaiting、_Gsyscall 和 _Gpreempted 几个状态；
- 可运行：Goroutine 已经准备就绪，可以在线程运行，如果当前程序中有非常多的 Goroutine，每个 Goroutine 就可能会等待更多的时间，即 _Grunnable；
- 运行中：Goroutine 正在某个线程上运行，即 _Grunning；


### 调度算法
**复用**
> 两种机制来实现对 goroutine 的复用，减少频繁创建、销毁带来的开销
- Work stealing 会从其他 P 的 G 队列中偷取 G
- hand off 则在当系统调用阻塞之时，释放绑定的P，交给其他空闲的线程执行

**并行**
> 多个 P 来并行地运行 goroutine

**抢占**
> 基于信号的抢占式调度主要解决了垃圾回收和栈扫描时存在的问题


## 实践中的常见问题

**GMP 中 P 的数量**
> https://www.jianshu.com/p/1a50330adf1b

在go中，线程的数量不受 CPU 核心数限制，能在系统调用中被阻塞的线程可以被大量创建
* Go 项目很多的并发其实是网络 IO；这种场景，由于 Go 封装的 net/http 库底层是基于 epoll 机制的，在建立连接以后并不会阻塞线程，所以增加GOMAXPROCS是没有效果的
* 对于其他会阻塞线程的 I/O 密集型（比如文件IO）或者系统调用比较多的场景，通过把GOMAXPROCS的值适当地增大到 CPU核心数以上，实际上可以提高系统的吞吐性能
 - IO密集型配置2N。
 - 其他情况或者 CPU 密集型，直接使用默认值N。


**goroutine 数量**
Go 的并发工具
1. Goroutine pool
2. Ants




**goroutine 泄漏与避免**
> 内存泄露指的是程序运行过程中已不再使用的内存，没有被释放掉，导致这些内存无法被使用，直到程序结束这些内存才被释放的问题

goroutine 泄漏即，存在 goroutine 从创建后一直运行到进程结束而没有符合预期地退出，一直占有资源。

要知道什么时候会发生协程泄漏，我们首先得知道协程怎样终止：
1. 正常执行完goroutine内的业务逻辑代码块
2. 发生了没有处理的panic
3. 被其他协程终止

当以上三点都没有满足，goroutine 发生泄漏，通常原因有：
1. channel 发生了阻塞
2. goroutine 中逻辑死循环
3. goroutine 中逻辑进入长时间阻塞等待

Eg: 在使用一个无缓冲通道errChan的时候，往通道里发送的数据"func1 err occured"只有在被读取后，发送才会完成



## 任务调度管理器

使用任务调度框架的主要目标包括如下几点：
- 处理依赖关系
- 处理超时
- 中断管理



## 开发中如何使用并发工具？

- 对于一些简单的场景（没有复杂的DAG依赖，没有任务的多次运行或者复用），可以直接使用 go func()加 sync.WaitGroup以及 chan 来解决并发问题
- 对于存在 DAG 依赖的场景，可以使用 JobMgr 或 ef_flow 来完成。
任务调度框架比较 
  - 对于想实现job复用、隐式依赖的情况，使用jobmgr
  - 对于任务复杂度小，简单实现并发的情况，使用 ef_flow


----



# `sync.WaitGroup`
> 在每一个goroutine启动时加一，在goroutine退出时减一。这需要一种特殊的计数器，这个计数器需要在多个goroutine操作时做到安全并且提供在其减为零之前一直等待的一种方法


`Add`和`Done`方法的不对称
* `Add`是为计数器加一，必须在worker goroutine开始之前调用，而不是在goroutine中,否则的话我们没办法确定Add是在"closer" goroutine调用Wait之前被调用
* `Done`却没有任何参数；其实它和`Add(-1)`是等价的



# 并发的退出

Go语言并没有提供在一个goroutine中终止另一个goroutine的方法，由于这样会导致goroutine之间的共享变量落在未定义的状态上

#### 退出一个的goroutine的方案
* channel里发送了一个简单的值，在countdown的goroutine中会把这个值理解为自己的退出信号

#### 想要退出两个或者任意多个goroutine
* 关闭了一个channel并且被消费掉了所有已发送的值，操作channel之后的代码可以立即被执行，并且会产生零值
* 不要向channel发送值，而是用关闭一个channel来进行广播




--------
# 基于共享变量的并发

## 竞争条件
> 程序在多个goroutine交叉执行操作时，没有给出正确的结果


数据竞争：无论任何时候，只要有两个goroutine并发访问同一变量，且至少其中的一个是写操作的时候就会发生数据竞争


三种方式可以避免数据竞争
1. 不要去写变量
2. 避免从多个goroutine访问变量 ->"不要使用共享数据来通信；使用通信来共享数据"
3. 允许很多goroutine去访问变量，但是在同一个时刻最多只有一个goroutine在访问。这种方式被称为“互斥”


即使当一个变量无法在其整个生命周期内被绑定到一个独立的goroutine时

1. 在一条流水线上的goroutine之间共享变量是很普遍的行为，在这两者间会通过channel来传输地址信息。
2. 如果流水线的每一个阶段都能够避免在将变量传送到下一阶段后再去访问它，那么对这个变量的所有访问就是线性的。
3. 其效果是变量会被绑定到流水线的一个阶段，传送完之后被绑定到下一个，以此类推。这种规则有时被称为串行绑定

```
func teller() {
    var balance int // balance is confined to teller goroutine
    for {
        select {
        case amount := <-deposits:
            balance += amount
        case balances <- balance:
        }
    }
}
```

## 竞争条件检测

go build，go run或者go test命令后面加上-race的flag
就会使编译器创建一个你的应用的“修改”版或者一个附带了能够记录所有运行期对共享变量访问工具的test，并且会记录下每一个读或者写共享变量的goroutine的身份信息。



## `sync.Mutex`互斥锁

binary semaphore:用一个容量只有1的channel来保证最多只有一个goroutine在同一时刻访问一个共享变量,一个只能为1和0的信号量

```
var (
    sema    = make(chan struct{}, 1) // a binary semaphore guarding balance
    balance int
)

func Deposit(amount int) {
    sema <- struct{}{} // acquire token
    balance = balance + amount
    <-sema // release token
}

func Balance() int {
    sema <- struct{}{} // acquire token
    b := balance
    <-sema // release token
    return b
}
```


临界区: 锁的持有者在其他goroutine获取该锁之前需要调用Unlock
```
func Deposit(amount int) {
    mu.Lock()
    balance = balance + amount
    mu.Unlock()
}
```

## `sync.RWMutex`读写锁
> 允许多个只读操作并行执行，但写操作会完全互斥。这种锁叫作“多读单写”锁（multiple readers, single writer lock）

```
var mu sync.RWMutex
var balance int64


func Balance() int {
	mu.RLock() // readers lock
	defer mu.RUnlock()
	return balance
}
```

* 调用了`RLock`和`RUnlock`方法来获取和释放一个读取或者共享锁。
* 调用`mu.Lock`和`mu.Unlock`方法来获取和释放一个写或互斥锁
* `RLock`只能在临界区共享变量没有任何写入操作时可用



## `sync.Once` 惰性初始化

一次性的初始化需要一个互斥量mutex和一个boolean变量来记录初始化是不是已经完成了；互斥量用来保护boolean变量和客户端数据结构
```
var loadIconsOnce sync.Once
var icons map[string]image.Image
// Concurrency-safe.
func loadIcons() {
    icons = map[string]image.Image{
        "spades.png":   loadIcon("spades.png"),
        "hearts.png":   loadIcon("hearts.png"),
        "diamonds.png": loadIcon("diamonds.png"),
        "clubs.png":    loadIcon("clubs.png"),
    }
}

func Icon(name string) image.Image {
    loadIconsOnce.Do(loadIcons)
    return icons[name]
}
```
1. 在第一次调用时，boolean变量的值是false, Do会调用loadIcons并会将boolean变量设置为true。
2. 随后的调用什么都不会做，但是mutex同步会保证loadIcons对内存（这里其实就是指icons变量啦）产生的效果能够对所有goroutine可见

## `Goroutines`和线程
### 动态栈
每一个OS线程都有一个固定大小的内存块（一般会是2MB）来做栈,这个栈会用来存储当前正在被调用或挂起（指在调用其它函数时）的函数的内部变量

一个`goroutine`会以一个很小的栈开始其生命周期
* 一般只需要2KB。一个goroutine的栈，和操作系统线程一样，会保存其活跃或挂起的函数调用的本地变量
* 和OS线程不太一样的是，一个goroutine的栈大小并不是固定的,栈的大小会根据需要动态地伸缩。而goroutine的栈的最大值有1GB，比传统的固定大小的线程栈要大得多


### Goroutine调度

OS线程会被操作系统内核调度: 保存一个用户线程的状态到内存，恢复另一个线程的到寄存器，然后更新调度器的数据结构
1. 一个硬件计时器会中断处理器，这会调用一个叫作scheduler的内核函数
2. 这个函数会挂起当前执行的线程并将它的寄存器内容保存到内存中，检查线程列表并决定下一次哪个线程可以被运行,并从内存中恢复该线程的寄存器信息，然后恢复执行该线程的现场并开始执行线程
3. 因为操作系统线程是被内核所调度，所以从一个线程向另一个“移动”需要完整的上下文切换

### `GOMAXPROCS`
> Go的调度器使用了一个叫做GOMAXPROCS的变量来决定会有多少个操作系统的线程同时执行Go的代码

其默认的值是运行机器上的CPU的核心数
```
for {
    go fmt.Print(0)
    fmt.Print(1)
}

$ GOMAXPROCS=1 go run hacker-cliché.go
111111111111111111110000000000000000000011111...

$ GOMAXPROCS=2 go run hacker-cliché.go
010101010101010101011001100101011010010100110...
```

### Goroutine没有ID号



Go调度器并不是用一个硬件定时器，而是被Go语言“建筑”本身进行调度的。
* 例如当一个`goroutine`调用了`time.Sleep`，或者被channel调用或者mutex操作阻塞时，调度器会使其进入休眠并开始执行另一个goroutine，直到时机到了再去唤醒第一个goroutine。
* 这种调度方式不需要进入内核的上下文，所以重新调度一个goroutine比调度一个线程代价要低得多
