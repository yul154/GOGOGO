* 控制程序的生命周期
* 管理可复用的资源池
* 创建可以处理任务的`goroutine`池

----
# `runner`
> 用于展示如何使用通道来监视程序的执行时间

```
import (
	"errors"
	"os"
	"os/signal"
	"time"
)

//Runner 在给定的超时时间内执行一组任务
// 并且在操作系统发送中断信号时结束这些任务
type Runner struct {
	//从操作系统发送信号
	interrupt chan os.Signal
	//报告处理任务已完成
	complete chan error
	//报告处理任务已经超时
	timeout <-chan time.Time
	//持有一组以索引为顺序的依次执行的以 int 类型 id 为参数的函数
	tasks []func(id int)
}

//统一错误处理

// ErrTimeOut 超时错误信息，会在任务执行超时时返回
var ErrTimeOut = errors.New("received timeout")

// ErrInterrupt 中断错误信号，会在接收到操作系统的事件时返回
var ErrInterrupt = errors.New("received interrupt")

//New 函数返回一个新的准备使用的 Runner，d：自定义分配的时间
func New(d time.Duration) *Runner {
	return &Runner{
		interrupt: make(chan os.Signal, 1),
		complete:  make(chan error),
		//会在另一线程经过时间段 d 后向返回值发送当时的时间。
		timeout: time.After(d),
	}
}

//Add 将一个任务加入到 Runner 中
func (r *Runner) Add(tasks ...func(id int)) {
	r.tasks = append(r.tasks, tasks...)
}

//Start 开始执行所有任务，并监控通道事件
func (r *Runner) Start() error {
	//监控所有的中断信号
	signal.Notify(r.interrupt, os.Interrupt)

	//使用不同的 goroutine 执行不同的任务
	go func() {
		r.complete <- r.run()
	}()

	//使用 select 语句来监控 goroutine 的通信
	select {
	//等待任务完成
	case err := <-r.complete:
		return err
	//任务超时
	case <-r.timeout:
		return ErrTimeOut
	}
}

//执行每一个已注册的任务
func (r *Runner) run() error {
	for id, task := range r.tasks {
		//检测操作系统的中断信号
		if r.gotInterrupt() {
			return ErrInterrupt
		}
		//执行已注册的任务
		task(id)
	}
	return nil
}

//检测是否收到了中断信号
func (r *Runner) gotInterrupt() bool {
	select {
	//当中断事件被触发时
	case <-r.interrupt:
		//停止接收后续的任何信号
		signal.Stop(r.interrupt)
		return true
		//继续执行
	default:
		return false
	}
}
```

* 程序可以在分配的时间内完成工作，正常终止；
* 程序没有及时完成工作，“自杀”；
* 接收到操作系统发送的中断事件，程序立刻试图清理状态并停止工作。

```
type Runner struct {
	//从操作系统发送信号
	interrupt chan os.Signal
	//报告处理任务已完成
	complete chan error
	//报告处理任务已经超时
	timeout <-chan time.Time
	//持有一组以索引为顺序的依次执行的以 int 类型 id 为参数的函数
	tasks []func(id int)
}
```
* `chan time.Time` : 这个通道用来管理执行任务的时间
* `os.Signal`接口抽象了不同操作系统上捕获和报告信号事件的具体实现
---
# `pool`
> 何使用有缓冲的通道实现资源池,，来管理可以在任意数量的goroutine之间共享及独立使用的资源。

`sync.pool`

```
 var studentPool = sync.Pool{
      New :func() interface{}{
            return new(Student)
      }
 }
 
 stu := studentPool.Get().(*Student)
 json.Unmarshal(buf,stu)
 studentPool.Put(stu)
```
----
# `work`
> 何使用无缓冲的通道来创建一个 goroutine池, 这些 goroutine 执行并控制一组工作，让其并发执行


```
type Worker interface {
    Task()
}

// Pool 提供一个 goroutine 池，这个池可以完成
// 任何已提交的 Worker 任务
type Pool struct {
    work chan Worker
    wg   sync.WaitGroup
}

func New(maxGoroutines int) *Pool {
    p := Pool{
        work: make(chan Worker),
    }

    p.wg.Add(maxGoroutines)
    for i := 0; i < maxGoroutines; i++ {
        go func() {
            for w := range p.work {
                w.Task()
            }
            p.wg.Done()
        }()
    }

    return &p
}
```

* .New方法中的for range 循环会一直阻塞，直到从 work 通道收到一个 Worker 接口值
* Shutdown 方法做了两件事
1. 关闭了 work 通道，这会导致所有池里的 goroutine 停止工作，并调用 WaitGroup 的 Done 方法
2. Shutdown 方法调用WaitGroup 的 Wait 方法，这会让 Shutdown 方法等待所有 goroutine 终止
---

