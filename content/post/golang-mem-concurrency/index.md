---
title: 【Golang进阶】内存同步模型
draft: false
date: 2021-03-24
categories: 
    - Go
---

## CSP 模型

Go 中相比于 Java 采用了 CSP（Communicating Sequential Process）并发模型，在消息传递模型中，并行进程通过消息传递相互交换数据。这种通信可以是异步的，就是说消息可以在接收者准备好之前发出；或是同步的，就是说消息发出前接收者必须准备好。同时 Go 基于 CSP 模型构建了一套 Happens Before 的可见性规则。

程序在多个 goroutine 中进行并发读写操作，必须序列化这些操作。可以通过 `channel` 或者其他同步原语（`sync` 包下的方法）来进行同步。


## Happens Before

在单个 goroutine 中，读写操作必须像代码中声明的顺序那样执行。在单个 goroutine 内，只有当指令重排序不会改变 golang 的语言规范时，编译器和处理器才能对指令进行重排序。由于重排序的存在，指令执行的顺序在不同的 goroutine 中观察的顺序也可能不一致。比如说：一个 goroutine 执行代码 `a=1; b=2` ，其他的 goroutine 可能先观察到 `b` 的修改，再观察到 `a` 的修改。

为指定数据读写的顺序，golang 中定义 happens before 原则。如果事件 e1 happens before 事件 e2，那么我们可以说 e2 happens after 事件 e1。如果 e1 没有 happens before e2，并且 e1 也没有 happens after e2，那么 e1 和 e2 就是并发的。

单个 goroutine 中，happens-before 顺序与代码中的顺序一致。当存在多个 goroutine 访问共享变量时，必须使用同步化事件来构建 happen-before 条件，以确保能读取到期望的写入。

> 规则1：单个 goroutine 中，happens-before 顺序与代码中的顺序一致

在 go 中变量 v 会被初始化该类型的 0 值，这个过程和一次写入是一样的。**读写超过机器字长的值会被拆分为多次操作，每次操作一个机器字长，并且这些操作是没有特定顺序的**。

### init

程序初始化是运行在单个 goroutine 中，但可能在该过程中会创建其他 goroutine 并行执行。

> 规则2：如果包 p 引入包 q，那么 q 的 `init` 函数 happen before 于任何 p 中的 `init`

### goroutine 创建

> 规则3：开启新 goroutine 的 go 语句 happen before 该 gorotine 的执行开始

```go
var a string
func f() {
    print(a)
}
func hello() {
    a = "hello, world"
    go f()
}
```
调用 hello 方法会在未来某个时间点（可能 hello 方法已返回）打印 “hello, world”

### goroutine 销毁

不能保证 goroutine 的退出会发生于程序中任意事件之前。

```go
var a string
func hello() {
    go func() { a = "hello" }()
    print(a)
}
```

上述的赋值操作没有进行同步化，所以它不保证能观察到其他 goroutine 的操作。事实上，激进的编译器可能直接删除这条 go 语句。

如果需要一个 goroutine 必须观察到其他 goroutine ，使用例如锁或 channel 的同步化机制来建立关联顺序。

### channel

Go 提倡在多个 goroutine 之间使用 channel 进行同步，在特定 channel 上的每个发送都会匹配一个读取，通常是在不同的 goroutine 中。

> 规则4：一个 channel 上第 k 个写操作，发生于第 k 个读操作完成之前

```go
var c = make(chan int, 10)
var a string
func f() {
    a = "hello, world"
    c <- 0
}
func main() {
    go f()
    <-c
    print(a)
}
```

上述代码保证输出 `hello, world`。对 a 的写操作 happens before 发送到 c ，发送到 c happens before 相对应 c 的接收，c 的接收 happens before print 方法。

> 规则5：对 channel 的关闭操作发生于接收端收到 0 值（因为 channel 被关闭）之前

在上面的例子中，把 `c<-0` 替换为 `close(c)` 有同样的效果


#### 无缓冲channel

在无缓冲 channel 上的读取操作发生于 channel 上写操作完成之前。

```go
var c = make(chan int) //无缓冲 channel
var a string
func f() {
    a = "hello, world"
    <-c
}
func main() {
    go f()
    c <- 0
    print(a)
}
```

上述代码也保证输出 `hello, world`。对 a 的写入操作发生于对 c 的读取之前，发生于对 c 的写入完成之前，发生于 `print` 之前。如果上述的 channel 是带缓冲的，那么该程序不能保证输出 `hello, world`（程序可能输出空字符串、崩溃或其他）

> 规则6：在一个容量为 C 的 channel 上第 k 个读取操作，发生于第 k+C 个写操作完成之前。

该原则将上一个原则推广到缓冲 channel 上。它允许通过缓冲 channel 对计数信号量进行建模：channel 中的元素个数对应信号量可用个数，channel 的容量对应信号量可同时使用的个数，写入一个元素会获取一个信号量，读取一个元素会释放一个信号量。

下面的代码会为每个工作列表开启一个 goroutine ，但 goroutine 中使用同一个 limit channel 来协调，以确保同一时间最多有 3 个函数在执行。

```go
var limit = make(chan int, 3)
func main() {
    for _, w := range work {
        go func(w func()) {
            limit <- 1
            w()
            <-limit
        }(w)
    }
    select{}
}
```

### 锁

sync 包有两种锁的实现，分别是 `sync.Mutex` 和 `sync.RWMutex`

> 规则7：对于任意 `sync.Mutex` 或者 `sync.RWMutex` 类型的对象 `l`，`l.Unlock` 发生于 `l.Lock` 返回之前

```go
var l sync.Mutex
var a string
func f() {
    a = "hello, world"
    l.Unlock()
}
func main() {
    l.Lock()
    go f()
    l.Lock()
    print(a)
}
```

上述代码保证输出 `hello, world`，第一次 `l.Unlock`（`f` 中）发生于第二次` l.Lock`（`main` 中）返回之前，`l.Lock` 的返回发生于 `print` 之前。

对任意 `sync.RWMutex` 变量 `l` 的调用 `l.RLock` ，第 n 次的 `l.RLock` 发生在第 n 次的 `l.Unlock` 之后，对应的 `l.RUnlock` 发生于 n+1 次的 `l.Lock` 之前

### Once

sync 包提供了一种安全的机制来初始化可能被多个 goroutine 使用的单例对象。允许多个线程同时执行 `once.Do(f)` ，但是只有一个线程能执行 `f()` ，其他的调用会阻塞直到 `f()` 返回。

> 规则8：`once.Do(f)` 中执行的 `f()` 发生于任意 `once.Do(f)` 返回之前

```go
var a string
var once sync.Once
func setup() {
    a = "hello, world"
}
func doprint() {
    once.Do(setup)
    print(a)
}
func twoprint() {
    go doprint()
    go doprint()
}
```

调用 `twoprint` 只会有一次 `steup` 调用发生。`steup` 函数会在 `print` 之前完成。最终结果为 `hello, world`，并且输出两次

## atomic包

atomic 包提供基于内存共享的同步语义，提供低级的内存同步功能。除非是一些特殊的或者底层应用，go 官方更推荐使用 channel 或其他更高级的同步工具，而不是共享内存。

atomic 本身只提供原子性、可见性保证，**不提供 happen before 的保证**，这也就是 `atomic` 与 `mutex` 的根本区别。因此，相比 `mutex` `atomic` 的实现简单很多，性能上 `atomic` 也比 `mutex` 略胜一筹：

```bash
$ go test -benchtime=10000000x -bench=.
goos: linux
goarch: amd64
pkg: demo/benchmark
BenchmarkAtomic-8   	10000000	       156 ns/op
BenchmarkMutex-8    	10000000	       526 ns/op
PASS
ok  	demo/benchmark	6.826s
```