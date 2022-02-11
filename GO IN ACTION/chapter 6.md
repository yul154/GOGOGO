* 使用 goroutine 运行程序
* 检测并修正竞争状态
* 利用通道共享数据

-----
# 并发与并行

Go 语言里的并发指的是能让某个函数独立于其他函数运行的能力
* 当一个函数创建为`goroutine`时，Go 会将其视为一个独立的工作单元,这个单元会被调度到可用的逻辑处理器上执行
* Go语言的并发同步模型来自一个叫作通信顺序进程（Communicating Sequential Processes，CSP）
  * 通过在`goroutine`之间传递数据来传递消息，而不是对数据进行加锁来实现同步访问。
  * 用于在`goroutine`之间同步和传递数据的关键数据类型叫作通道(channel）
  * 操作系统会在物理处理器上调度线程来运行，而 Go 语言的运行时会在逻辑处理器上调度goroutine来运行。每个逻辑处理器都分别绑定到单个操作系统线程


thread VS process
* 当运行一个应用程序（如一个 IDE 或者编辑器）的时候，操作系统会为这个应用程序启动一个进程。
    * 可以将这个进程看作一个包含了应用程序在运行中需要用到和维护的各种资源的容器
* 一个线程是一个执行空间，这个空间会被操作系统调度来运行函数中所写的代码


`goroutine`
1. 如果创建一个`goroutine`并准备运行，这个`goroutine`就会被放到调度器的全局运行队列中
2. 调度器就将这些队列中的`goroutine`分配给一个逻辑处理器，并放到这个逻辑处理器对应的本地运行队列
3. 本地运行队列中的 goroutine 会一直等待直到自己被分配的逻辑处理器执

![截屏2022-02-09 下午6 38 16](https://user-images.githubusercontent.com/27160394/153181291-cd5e30b0-dbbd-41b4-bf57-ff4ffc81eb94.png)

被阻塞的`goroutine`
1. 线程和`goroutine`会从逻辑处理器上分离，该线程会继续阻塞，等待系统调用的返回
2. 这个逻辑处理器就失去了用来运行的线程。所以，调度器会创建一个新线程，并将其绑定到该逻辑处理器上
3. 调度器会从本地运行队列里选择另一个`goroutine`来运行。
4. 一旦被阻塞的系统调用执行完成并返回，对应的`goroutine`会放回到本地运行队列，


并发（concurrency）不是并行（parallelism）
* 并行是让不同的代码片段同时在不同的物理处理器上执行。并行的关键是同时做很多事情，
* 并发是指同时管理很多事情，这些事情可能只做了一半就被暂停去做别的事情了。

-----
# `goroutine`

```
package main
import (
  "fmt"
  "runtime"
  "sync"
)

func main(){

    runtime.GOMAXPROCS(1) //允许程序更改调度器可以使用的逻辑处理器的数量
    var wg sync.WaitGroup
    wg.Add(2)
    fmt.Println("Start Goroutines")
    
    go.func(){
      
       defer wg.Done() // 退出时调用 Done 方法
       for count := 0; count < 3; count++(
       
            for char := 'A'; char < 'A'+26； char++（
            
                fmt.Printf("%c",char)
            
            ）
       )
    
    }()
    
    fmt.Println("Waiting To Finish")
    wg.wait()
    fmt.Println("\nTerminating Program")

}
```
----
# 竞争状态
> 两个或者多个`goroutine`在没有互相同步的情况下，访问某个共享的资源


* 调用了`runtime`包的`Gosched`函数，用于将`goroutine`从当前线程退出

```
go build -race // 检测竞争标志
```
一种修正代码、消除竞争状态的办法是，使用 Go 语言提供的锁机制，来锁住共享资源，从而保证 goroutine 的同步状态
----
# 锁住共享资源

## 原子函数
```
func main() {

 wg.Add(2) // 创建两个 goroutine
 go incCounter(1)
 go incCounter(2)

 wg.Wait()

fmt.Println("Final Counter:", counter)
}
 func incCounter(id int) {
  defer wg.Done()
  for count := 0; count < 2; count++ {
    atomic.AddInt64(&counter, 1) //原子操作
    runtime.Gosched()
    }
 }
```

## 互斥锁
```
mutex.Lock()
{

   value := counter
   runtime.Gosched()
   value++
   counter = value

}
mutex.Unlock()
```

## 通道

当一个资源需要在 goroutine 之间共享时，通道在 goroutine 之间架起了一个管道，并提供了确保同步交换数据的机制

```
unbuffered := make(chan int)
buffered := make(chan string, 10)
buffered <- "Gopher"
value := <-buffered
```
* 第一个参数需要是关键字 chan，
* 之后跟着允许通道交换的数据的类型
* 如果创建的是一个有缓冲的通道，之后还需要在第二个参数指定这个通道的缓冲区的大小
* 向通道发送值或者指针需要用到`<-`操作符

### 无缓冲的通道
> 指在接收前没有能力保存任何值的通道

* 这种类型的通道要求发送 goroutine 和接收 goroutine 同时准备好
* 如果两个`goroutine`没有同时准备好，通道会导致先执行发送或接收操作的 goroutine 阻塞等待
* 通道进行发送和接收的交互行为本身就是同步的

```
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

var wg sync.WaitGroup

func init() {
	rand.Seed(time.Now().UnixNano())
}

func main() {
	court := make(chan int)
	wg.Add(2)

	go player("a", court)
	go player("b", court)

	court <- 1
	wg.Wait()

}

func player(name string, court chan int) {

	defer wg.Done()

	for {

		ball, ok := <-court
		if !ok {

			fmt.Printf("Player %s Won\n", name)
			return

		}

		n := rand.Intn(100)
		fmt.Printf("%d\n", n)

		if n%13 == 0 {

			fmt.Printf("Player %s Miss \n", name)
			close(court)
			return
		}

		fmt.Printf("Player %s Hit %d\n", name, ball)
		ball++
		court <- ball

	}

}

```

```
package main
import (
    "fmt"
    "sync"
    "time"
)
// wg 用来等待程序结束
var wg sync.WaitGroup
// main 是所有Go 程序的入口
func main() {
    // 创建一个无缓冲的通道
    baton := make(chan int)
    // 为最后一位跑步者将计数加1
    wg.Add(1)
    // 第一位跑步者持有接力棒
    go Runner(baton)
    // 开始比赛
    baton <- 1
    // 等待比赛结束
    wg.Wait()
}
// Runner 模拟接力比赛中的一位跑步者
func Runner(baton chan int) {
    var newRunner int
    // 等待接力棒
    runner := <-baton
    // 开始绕着跑道跑步
    fmt.Printf("Runner %d Running With Baton\n", runner)
    // 创建下一位跑步者
    if runner != 4 {
        newRunner = runner + 1
        fmt.Printf("Runner %d To The Line\n", newRunner)
        go Runner(baton)
    }
    // 围绕跑道跑
    time.Sleep(100 * time.Millisecond)
    // 比赛结束了吗？
    if runner == 4 {
        fmt.Printf("Runner %d Finished, Race Over\n", runner)
        wg.Done()
        return
    }
    // 将接力棒交给下一位跑步者
    fmt.Printf("Runner %d Exchange With Runner %d\n",
        runner,
        newRunner)
    baton <- newRunner
}
```

### 有缓冲的通道
> 在被接收前能存储一个或者多个值的通道

* 只有在通道中没有要接收的值时，接收动作才会阻塞
* 只有在通道没有可用缓冲区容纳被发送的值时，发送动作才会阻塞。

```

```


当通道关闭后，goroutine 依旧可以从通道接收数据，但是不能再向通道里发送数据
* 这允许通道关闭后依旧能取出其中缓冲的全部值，而不会有数据丢失 

> 为什么Go语言对通道要限制长度而不提供无限长度的通道？

当提供数据一方的数据供给速度大于消费方的数据处理速度时，如果通道不限制长度，那么内存将不断膨胀直到应用崩溃
* 限制通道的长度有利于约束数据提供方的供给速度
* 供给数据量必须在消费方处理量+通道长度的范围内，才能正常地处理数据。

