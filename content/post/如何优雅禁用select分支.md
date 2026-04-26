+++
date = '2026-04-26T22:56:18+08:00'
draft = false
title = '如何优雅禁用select分支'
+++


Go语言并发中，select语句像一个通信指挥官能够协调多个通道（channel）的通信，它同时监听多个通道，一旦某个通道有数据就绪，就立刻执行对应的操作。


但实际工程中，可能需要按照具体情况来控制某个 channel 是否参与调度，禁用掉不再需要的通道，从而让它不再参与 select 的调用。


碰到禁用/启用某个功能或模块，就直觉的想到通过增加变量，或者 if 判断来解决。但控制 channel 的调用，Go 语言提供了一个优雅且地道的原生解决方法：nil channel。


今天，我们就来看看，如何利用 nil channel 来实现 select 分支的优雅启停。

#### Nil Channel
Go 语言中，通过 var 或者 make 来声明 channel

``` go

var ch chan int

```

或

``` go

 ch := make(chan int)

```

区别在于得到的 channel 值不同。通过 var 声明的 channel 值为 nil，也就是Nil channel。

对一个Nil channel 进行操作，会发生什么呢？

给定一个 Nil channel：
- <- ch 永久阻塞当前 Goroutine
- ch <- v 永久阻塞当前 Goroutine
- close(ch)  close 引发 panic

其实，读写一个 Nil channel 还可能会引起死锁。

```go
func main() {
	var ch chan int

	// 读数据阻塞
	for v := range ch {
		fmt.Println("output value:", v)
	}
}
```
运行上面代码
```
all goroutines are asleep - deadlock!
```
产生这种死锁 go 运行时还可以及时发现，自动触发保护机制，主动终止程序并打印错误信息。更糟的是还可能引起另外一种死锁---“静默死锁”，引发 goroutine 泄漏。

Nil channel 处处是陷阱，就一点用都没有？

凡事有弊必有利，情花毒的解药就是离它不远的断肠草。

#### 一个例子

``` go
func genChan(vs ...int) chan int {
	c := make(chan int)
	go func() {
		for _, v := range vs {
			c <- v
			time.Sleep(time.Duration(rand.Intn(1000)) * time.Millisecond)
		}
		close(c)
	}()

	return c
}

func main() {

	ch1 := genChan(1, 2, 3)
	ch2 := genChan(4, 5, 6)
	ch3 := make(chan int)

	go func() {
		for {
			select {
			case v := <-ch1:
				ch3 <- v

			case v := <-ch2:
				ch3 <- v
			}
		}
	}()

	for v := range ch3 {
		fmt.Println("ch3:", v)
        time.Sleep(1 * time.Second)
	}
}
```
这个例子有两个函数 genChan 和 main。 genChan 接受一个可变整形参数，用来生成一个通道。

main 函数启动一个协程通过 select 语句读取两个通道的值并把结果汇聚到同一个通道 ch3 中，最后打印通道 ch3 中的结果。

运行程序，会输出：
```
ch3: 4
ch3: 1
ch3: 2
ch3: 3
ch3: 5
ch3: 6
ch3: 0
ch3: 0
```

结果中在输出了1-6几个数字之后，还输出了0。

仔细想一想这个 0 值也不难理解。在 genChan 中，生成一个新的通道之后，会用 close 方法关闭掉通道，在 go 中，当从一个关闭的通道中读取值时，会返回默认值，在 genChan 中，默认值是0。

可以使用

```
v, ok := ch
```
语句来检查通道是否已经被关闭，对于关闭的通道需要过滤掉，在真实的业务中可能是不需要处理这种值的。

```
	go func() {
		for {
			select {
			case v, ok:= <-ch1:
                if !ok {
                    continue
                }
				ch3 <- v

			case v, ok:= <-ch2:
                if !ok {
                    continue
                }
				ch3 <- v
			}
		}
	}()
```
通过修改，只有当通道中有值时，才会给通道 ch3 中写入，否则就跳过当前分支。

程序结果中不会再输出 0，当 ch1, ch2 两个通道被 close 之后，这个 for 迭代会一直运行，而且由于 select 语句再选择分支时是随机的，不能判断两个通道的完成顺序从而提前终止某个通道。

程序陷入了死循环，造成忙循环。

#### 禁用 select 分支
对于已关闭的通道，go 运行时依然可以对 select 语句中的分支执行计算并且认为该通道是可读状态。

有没有办法在通道被关闭之后，修改通道的状态，当运行时碰到这种状态跳过 select 分支的执行呢？

在 go 运行时中，通过将通道的值置为 nil 来禁用某个分支，就是之前提到的 Nil channel。

再次修改代码：

```
go func() {
    defer close(ch3)

    for {
        if ch1 == nil && ch2 == nil {
            break
        }
        select {
        case v, ok := <-ch1:
            if !ok {
                ch1 = nil
                continue
            }
            ch3 <- v
        case v, ok := <-ch2:
            if !ok {
                ch2 = nil
                continue
            }
            ch3 <- v
        }
    }
}()
```
在上面修改中，做了两处修改：
- 进入 for 迭代中，首先检查通道的值是否都为 nil,都为 nil 就退出迭代
- 当通道被关闭之后，把该通道的状态从 close 状态切换到 nil 状态

运行程序，打印完 1-6 几个数字之后，正常结束。
#### 总结
Nil Channel 是 go 运行时对 select 分支的特殊处理，通过对通道状态的修改来禁用当前分支，利用它可以简单优雅的实现并发编程。

同时也要注意 Nil Channel 的陷阱。

实践中，当需要禁用 select 分支中某个或者多个的时候，就需要使用 Nil Channel，其他情况下，在使用channel之前，最好使用 make 来初始化，否则可能会调入陷阱中。

