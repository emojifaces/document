# Channel
Go语言的并发模型是CSP（Communicating Sequential Processes），提倡通过通信共享内存而不是通过共享内存而实现通信。
如果说goroutine是Go程序并发的执行体，channel就是它们之间的连接。channel是可以让一个goroutine发送特定值到另一个goroutine的通信机制。

## channel 类型
channel是一种类型，一种引用类型，空值是nil。声明通道类型的格式如下：

```go
var 变量 chan 元素类型

var ch1 chan int   // 声明一个传递整型的通道
var ch2 chan bool  // 声明一个传递布尔型的通道
var ch3 chan []int // 声明一个传递int切片的通道
```
## 创建 channel
创建channel的格式如下：
```go
make(chan 元素类型, [缓冲大小])

ch4 := make(chan int)
ch5 := make(chan bool)
ch6 := make(chan []int)
```
## channel 操作
通道有发送（send）、接收(receive）和关闭（close）三种操作。

发送和接收都使用<-符号。
```go
ch := make(chan int)

// 发送
ch <- 10 // 把10发送到ch中

// 接收
x := <- ch // 从ch中接收值并赋值给变量x
<- ch // 从ch中接收值，忽略结果

// 关闭
close(ch)
```
只有在通知接收方goroutine所有的数据都发送完毕的时候才需要关闭通道。通道是可以被垃圾回收机制回收的，它和关闭文件是不一样的，在结束操作之后关闭文件是必须要做的，但关闭通道不是必须的。

关闭后的通道有以下特点：

1. 对一个关闭的通道再发送值就会导致panic。
2. 对一个关闭的通道进行接收会一直获取值直到通道为空。
3. 对一个关闭的并且没有值的通道执行接收操作会得到对应类型的零值。
4. 关闭一个已经关闭的通道会导致panic。

## 无缓冲通道
无缓冲的通道只有在有人接收值的时候才能发送值，否则会在执行时报错。

无缓冲通道上的发送操作会阻塞，直到另一个goroutine在该通道上执行接收操作，这时值才能发送成功，两个goroutine将继续执行。相反，如果接收操作先执行，接收方的goroutine将阻塞，直到另一个goroutine在该通道上发送一个值。

使用无缓冲通道进行通信将导致发送和接收的goroutine同步化。因此，无缓冲通道也被称为同步通道。

## 有缓冲通道
只要通道的容量大于零，那么该通道就是有缓冲的通道，通道的容量表示通道中能存放元素的数量。

使用内置的len函数获取通道内元素的数量，使用cap函数获取通道的容量
## for range从通道循环取值
当向通道中发送完数据时，我们可以通过close函数来关闭通道。

当通道被关闭时，再往该通道发送值会引发panic，从该通道取值的操作会先取完通道中的值，再然后取到的值一直都是对应类型的零值。

```go
func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)
	// 开启goroutine将0~100的数发送到ch1中
	go func() {
		for i := 0; i < 100; i++ {
			ch1 <- i
		}
		close(ch1)
	}()
	// 开启goroutine从ch1中接收值，并将该值的平方发送到ch2中
	go func() {
		for {
			i, ok := <-ch1 // 通道关闭后再取值ok=false
			if !ok {
				break
			}
			ch2 <- i * i
		}
		close(ch2)
	}()
	// 在主goroutine中从ch2中接收值打印
	for i := range ch2 { // 通道关闭后会退出for range循环
		fmt.Println(i)
	}
}
```
通常使用的是for range的方式判断该通道是否被关闭。使用for range遍历通道，当通道被关闭的时候就会退出for range。

## 单向通道
chan<- int是一个只写单向通道（只能对其写入int类型值），可以对其执行发送操作但是不能执行接收操作

<-chan int是一个只读单向通道（只能从其读取int类型值），可以对其执行接收操作但是不能执行发送操作。

## goroutine池
在工作中我们通常会使用可以指定启动的goroutine数量–worker pool模式，控制goroutine的数量，防止goroutine泄漏和暴涨。

一个简易的work pool示例代码如下：
```go
func worker(id int, jobs <-chan int, results chan<- int) {
	for j := range jobs {
		fmt.Printf("worker:%d start job:%d\n", id, j)
		time.Sleep(time.Second)
		fmt.Printf("worker:%d end job:%d\n", id, j)
		results <- j * 2
	}
}


func main() {
	jobs := make(chan int, 100)
	results := make(chan int, 100)
	// 开启3个goroutine
	for w := 1; w <= 3; w++ {
		go worker(w, jobs, results)
	}
	// 5个任务
	for j := 1; j <= 5; j++ {
		jobs <- j
	}
	close(jobs)
	// 输出结果
	for a := 1; a <= 5; a++ {
		<-results
	}
}
```