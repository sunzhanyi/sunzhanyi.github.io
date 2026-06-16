+++
date = '2026-05-25T07:12:06+08:00'
draft = false
title = 'Go优雅关闭:别只会调用Shutdown'
+++

在工作当中，可能都遇到过这些情况：辛辛苦苦制作的 PPT 没保存电脑就蓝屏、下载到 99% 的文件因断网而前功尽弃。问题的根源都是：系统或进程被瞬间强制终止，完全没有留出处理善后工作的时间，进入了**粗暴关闭**流程。

在 Go 程序开发中，直接调用 `os.Exit()` 或对 Go 程序执行 `kill -9 ` 也会引发：
- **defer不执行：** defer 是 Go 中经常被用来处理函数退出时的资源回收，比如锁释放，文件句柄关闭，数据库链接回收。粗暴退出，会直接绕过 defer，导致资源泄漏。
- **数据状态不一致：** 通常发生在执行数据库事务或分布式系统中，事务执行一半，连接突然中断导致事务无法提交和回滚，可能留下不完整的数据。
- **故障排查困难：** 由于粗暴关闭时，程序在安全退出时记录的日志信息不会被执行，在外界看来，程序就是突然没了，找不到任何线索。

粗暴关闭会给系统和 Go 程序带来很多后遗症，如何让程序正常退出并安全着陆，就需要一套更完善的退出机制：**优雅关闭（Graceful Shutdown）**

### 什么是优雅关闭（Graceful Shutdown）

优雅关闭并没有一个绝对统一的定义，它在不同的技术领域中，有着不同的关注点，本质上都是解决同一问题：系统在终止前有序地完成善后工作，而不是被粗暴中断。

在操作系统中，优雅关闭更多的像是 OS 和用户进程之间的**信号约定**。当收到 SIGTERM（15号信号） 或 SIGINT （ctrl + c）优雅退出信号时，进程需要按照约定完成资源清理然后退出，构成 OS 与 用户进程之间的通讯协议。

当这种机制演进到 Kubernetes 等云原生基础设施中时，优雅关闭是一套标准化的**Pod终止流程**。是云原生 OS 与容器应用之间的生命周期协议：
- Pod 被终止时，如何把 Pod 从服务端点（Endpoint）中安全摘除
- 通过 `terminationGracePeriodSeconds` 机制，约定了 Pod 内容器进程的退出节奏

最终，这些基础设施层面的约定都需要落实到应用代码中。在 Go 中，优雅关闭更是一套可执行的**生命周期管控闭环**，通常包含以下几个关键步骤：
- **接收终止信号（os.Signal）:** 监听并接收操作系统发出的退出指令，作为关闭流程的起点。
- **停止接收新工作（Shutdown）:** 关闭监听入口，拒绝新的请求、连接或任务进入系统。
- **完成在途工作（Drain）：** 等待已经开始处理的请求、连接或后台任务正常结束。
- **协调后台组件退出（Context / WaitGroup）:** 向 Goroutine、消费者、定时任务等后台执行单元传播退出信号，并等待其完成清理。
- **超时强制终止（Timeout）:** 如果超过预设时间仍未完成退出，则执行强制关闭，避免进程无限期阻塞。

因此，Go 中的优雅关闭，是在收到终止信号后，停止接收新工作、完成在途工作、协调后台任务退出，并在有限时间内完成整个应用生命周期的有序流程。

### 如何实现优雅关闭

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	if err := run(); err != nil {
		log.Fatal(err)
	}
}

func run() error {
	server := createServer()
	// 启动 HTTP 服务
	serverErr := make(chan error, 1)
	go func() {
		defer close(serverErr)
		log.Println("server started on :8080")
		if err := server.ListenAndServe(); err != nil &&
			!errors.Is(err, http.ErrServerClosed) {
			serverErr <- err
		}
	}()
	// 捕获信号
	ctx, stop := signal.NotifyContext(
		context.Background(),
		os.Interrupt,
		syscall.SIGTERM,
	)
	defer stop()

	// 等待：
	// 1. 收到退出信号
	// 2. 服务异常退出
	select {
	case <-ctx.Done():
		log.Println("shutdown signal received")
	case err := <-serverErr:
		return err
	}

	// 给优雅关闭一个最大时间窗口
	shutdownCtx, cancel := context.WithTimeout(
		context.Background(),
		30*time.Second,
	)
	defer cancel()
	log.Println("graceful shutdown start")
	// 停止接收新请求
	// 等待在途请求完成
	if err := server.Shutdown(shutdownCtx); err != nil {
		log.Printf("graceful shutdown failed: %v", err)
		// Shutdown 超时后的最后兜底
		if closeErr := server.Close(); closeErr != nil {
			return errors.Join(err, closeErr)
		}
		return err
	}
	log.Println("graceful shutdown completed")
	return nil
}

func createServer() *http.Server {
	mux := http.NewServeMux()
	// 模拟耗时请求
	mux.HandleFunc("/sleep", func(w http.ResponseWriter, r *http.Request) {
		log.Println("request started")
		time.Sleep(10 * time.Second)
		log.Println("request finished")
		w.Write([]byte("done"))
	})
	return &http.Server{Addr: ":8080", Handler: mux}
}

```

代码中包含了三个函数。

`main` 函数是 Go 程序的入口，实现很简单，调用 `run` 函数，如果出错，打印错误信息然后退出。这样做能获取的直接好处是 `main` 函数作为入口点的可测试性，也是社区中推荐的写法。

`createServer` 用来创建一个 `http.Server` 对象，注册一个 `sleep` 路由。虽然和优雅关闭的实现没有关系，但通过它，可以感受到优雅关闭过程中真实业务退出的流程。

`run` 函数实现了三个功能：服务启动，信号注册，服务关闭。

#### 服务启动
服务启动过程有点特别，我们通过一个 `goroutine` 启动服务。因为 `server.ListenAndServe` 是阻塞方法，为了不阻塞主程序的运行，需要与主程序隔离，这也是 Go 语言处理网络服务高并发请求的标准工程实践。有意思的是，这里需要借用一个通道来传递启动错误给主程序。

```go
server := createServer()
// 启动 HTTP 服务
serverErr := make(chan error, 1)
go func() {
    defer close(serverErr)
    log.Println("server started on :8080")
    if err := server.ListenAndServe(); err != nil &&
        !errors.Is(err, http.ErrServerClosed) {
        serverErr <- err
    }
}()
```
#### 信号注册
信号注册是优雅关闭的基石。这里我们关注的是 `os.Interrupt` 和 `syscall.SIGTERM`，其中 `os.Interrupt` 是 `syscall.SIGINT` 的别名，通过这个别名，Go 语言为开发者屏蔽了底层操作系统之间处理信号的差异，提供统一的信号处理方法。

接下来程序等待退出和服务异常退出信号。当收到退出信号时，进入优雅关闭阶段，当收到服务异常退出信号时，进入 `main` 函数输出异常退出信息。

```go
// 捕获信号
ctx, stop := signal.NotifyContext(
    context.Background(),
    os.Interrupt,
    syscall.SIGTERM,
)
defer stop()

// 等待：
// 1. 收到退出信号
// 2. 服务异常退出
select {
case <-ctx.Done():
    log.Println("shutdown signal received")
case err := <-serverErr:
    return err
}
```
#### 服务关闭
如果主程序收到上一步注册的信号，会继续执行优雅关闭。通过调用 `server.Shutdown` 函数关闭服务：
- 关闭监听器：首先会关闭所有打开的Listeners，后续的新请求不会被接收
- 关闭空闲连接：接着关闭处于空闲中的连接
- 等待活跃连接：最后对于活跃的连接，Go 语言会无限期的等待，直到连接状态为空闲状态才执行关闭

这里需要注意的是，防止服务关闭出现无限期的等待，我们需要给优雅关闭加上一个超时限制。因此在实现中增加了 `context` 超时机制，当优雅关闭超过设定的时间30秒就会强制关闭服务而不是一直等待。

```go
// 给优雅关闭一个最大时间窗口
shutdownCtx, cancel := context.WithTimeout(
    context.Background(),
    30*time.Second,
)
defer cancel()
log.Println("graceful shutdown start")
// 停止接收新请求
// 等待在途请求完成
if err := server.Shutdown(shutdownCtx); err != nil {
    log.Printf("graceful shutdown failed: %v", err)
    // Shutdown 超时后的最后兜底
    if closeErr := server.Close(); closeErr != nil {
        return errors.Join(err, closeErr)
    }
    return err
}

log.Println("graceful shutdown completed")
return nil
```

#### 关闭后台任务
在实际项目中，我们的应用除了运行 HTTP Server 之外，还包含消息消费者，定时任务，后台worker这些长期运行的 Goroutine。 

对于这些后台任务，Go 语言并没有提供类似 `server.Shutdown` 的用法来停止它们，因此需要设计额外的退出机制。

一种比较简单的实现方式是通过 `channel` 来传递停止信号。当收到停止信号并完成 HTTP Server 退出之后通过 `close(stop)` 来退出后台任务。
```go
var wg sync.WaitGroup
stop := make(chan struct{})

wg.Add(1)
go func() {
	defer wg.Done()
	for {
		select {
		case <-stop:
			log.Println("worker stopped")
			return
		default:
			log.Println("worker running")
			time.Sleep(time.Second)
		}
	}
}()

close(stop)
wg.Wait()
```

### 总结
在 Go 语言中，`Shutdown` 函数是实现优雅关闭的一个重要环节，除此之外还需要我们处理操作系统的信号，控制优雅关闭超时时间，后台任务退出等其他过程。完成了这些工作之后，不断实践和完善这个架构，就能优雅的应对各种工作中的场景。
