这是一篇描述Go语言中多goroutine在对同一变量进行读写操作情况下如何保证逻辑一致性的教程，来自于Go官方语言2014年五月三十一日版本。

Go语言内存模型规定了在一个goroutine中一个变量的读取的情况下，确保能够观察到在其他另外goroutine中写入同样变量的值。也就是说，如果在多个goroutine操作修改同一个变量状态情况下，Go内存模型能够保证一个goroutine对变量写入的数据能够被其他goroutine正常读取，类似多线程编程中两个线程对同一个变量读写保证一样。

**建议**
   
   
   多个goroutine同时访问修改同一数据的编程必须是序列化访问。

　　为了实现序列化访问，使用channel操作或其他同步语法如sync和sync/atomic进行数据保护。

　　你必须阅读本文档所有部分才能理解你的程序运作机制，这样你就很聪明。

　　但是不要聪明。

**Happens Before**
　　
在单个goroutine中，读取和写入行为需要像它们在代码中编写的顺序一样，因为，当单个goroutine中这种读写操作重新排序不会改变其最终结果时，编译器和处理器也许会重排读取和写入的顺序。正是因为这种顺序重排，被一个goroutine观察到执行顺序也许就不同于同样代码在另外一个goroutine中执行顺序。举例：如果一个goroutine执行 a=1;b=2;，另外一个goroutine也许看到在a值更新之前b已经更新到2这个值了。

为了规定读取和写入，我们定义了happens before，这是Go语言中内存操作执行的偏序(partial order)，如果事件e1发生在事件e2之前，那么我们说事件e2发生在事件e1之后，也可以这么说，如果e1没有发生在e2之前，同时也不会发生在e2之后，那么我们说e1和e1是同时并发发生的。

在单个goroutine中，happens-before顺序是程序代码想要表达的顺序。

如果下面两个条件都满足，对于变量v的一个读操作r被允许观察到(之前)对v的写操作w。

r 没有发生在w之前(not happen before，r发生在w之后或同时发生)
并没有其他对v的写操作 w'发生在w之后与r之前（也就是说：在r和w之间再也没有除这两个操作以外的其他任何写操作）
为了保证变量v的一个读操作r能够获得对这个变量v的写操作，必须确保w是r能够看到的唯一的写操作，只有下面两个条件满足，r就能够确保观察到w：

w 发生在r之前(happens before).
任何对共享变量v的其他写操作或者发生在w之前，或者发生在r之后。（也就是：r和w之间没有任何其他写操作）
这一对条件强于先前第一对，它指定没有任何与w或r同时发生其他写操作。

在单个goroutine中，是不存在并发的。举例，下面两个语句前后是等价的：当多个goroutine访问同一共享变量v时，一个读操作r观察到被最近操作w对变量v的写入结果值，，它们必须使用同步事件来创建happen-before条件来确保读取到那个写操作的值。

对go中任意一个类型的零值初始化操作在内存模型中也看作是一个写入操作。

对大于一个机器字节的读写操作可看作多个顺序不定的机器字节(machine-word-sized)操作。

**同步**
**初始化**

　　程序初始化是运行在单个goroutine中，但是这个goroutine会创建同时其他goroutine运行。

　　如果一个包p导入了包q, 那么q的init函数的完成是发生在任何p的其他任何开始之前。

　　函数main.main的开始是发生在所有init函数完成了之后的。

**Goroutine创建**

　　注意，语句“go”会启动一个新的gorountine，这是发生在这个gorountine执行开始之前。举例如下：
  
``` go
var a string 
func f() {
     print(a)
} 
func hello() {
     a = "hello, world"
     go f()
}
```
　　调用hello函数不保证立即打印出"hello,world"，可能会在将来某个时刻(可能会在hello函数已经返回以后)打印出“hello, world”。

**Goroutine解构**

　　gorountine的退出并不保证其发生在程序中任何操作事件之前，比如：
  
``` go
var a string
func hello() {
     go func() { a = "hello" }()
     print(a)
}
```
 

a分配的值没有使用任何同步事件，这样它就不保证能被其他gorountine看到，实际上，攻击性强的编译器也许已经删除了整个“go func()....”语句

如果你希望一个gorountine必须被其他gorountine看到，那么使用同步机制，比如锁或channel通讯机制创建一个相对顺序。

**Channel通讯**

　　Channel通讯是gorountine之间主要的同步方法，发送到channel每个事件消息将被另外一个gorountine收到。

　　在一个channel中A发送将发生在从channel完全接受之前。案例：
``` go
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

这段代码将保证打印出"hello, world"，对a的写操作发生在对channel通道c的写操作之前，而通道c写操作会发生通道c的完成接受之前，通道c的接受完成是发生在print之前。

channel通道的关闭是发生在一个返回代表channel已经关闭的0值的接受操作之前。

在之前这个案例，可以使用close(c)代表 c <- 0，这会yield(线程用语)一个程序，同样保证操作行为顺序。

从一个无缓冲的channel中接受操作会发生该对channel发送操作完成之前。下面代码类似前面，但是使用了unbuffered无缓冲的channel：

``` go
var c = make(chan int)
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
 

这段代码也确保打印出“hello, world”，对a的写操作发生在对c的接受操作之前，而对c的接受操作发生在相应的对c发送操作完成之前，但是对c的发生操作完成肯定在print之前发生。

如果channel被缓冲了比如使用c = make(chan int, 1)那么程序将不会保证能打印出"hello,world"，它也许打印出空字符串或崩溃或其他什么事情会发生。

一个有容量C的通道第k大小接受操作发生在发送第第k+C个发送操作之前。

这个规则对于缓冲channel涵括之前的规则，这样能够允许一个计数semaphore 能被缓冲channel建模，channel中条目的数量对应于channel活动用户的数量，channel的容量对应于并发用户的最大数量，这些用户会在发送一个条目时获得semaphore，并且在接受到这个条目后释放semaphore，这是通常限制性并发的通用原理。

下面代码是在工作列表中每项启动一个goroutine，goroutine之间使用limit这个channel来确保一次最多运行三个函数。
``` go
var limit = make(chan int, 3)
 
func main() {
     for _, w := range work {
             go func() {
                    limit <- 1
                    w()
                    <-limit
             }()
     }
     select{}
}
```
