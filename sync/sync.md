# 概述
sync包对并发和同步机制进行了实现，但显然并发编程这样一个话题过于庞大，无法在一篇博客里面详细展开，所以本文的重点放在sync包的使用。

不过这里首先对并发的背景进行简单的介绍，在单线程的程序中，同一个时刻只存在一个线程对数据进行访问，访问永远是线性的，不需要额外的机制保障；但是当同时存在多个线程可能同时访问一个数据时，由于线程调度的特性，会带来难以预料的结果。试想如下代码会输出什么结果，正常情况会是3，但实际上可能的结果是2或者3，这是由于函数或者代码的运行没有原子性，比如在线程1执行Add1操作时被暂停了（已经读取data，还未写回），然后开始运行线程2，并读取数据，然后两个线程依次执行到结束，由于两个线程读取的值都是data=1，执行Add1后都得到2，而非3。

```
import (
      "fmt"
      "time"
)

func Add1(data *int) {
     tmp = *data
     *data = tmp + 1
}

func main(){
     data = 1
     go Add1(&data)
     go Add1(&data)
     time.Sleep(10)
     fmt.Print(data)
     
}
```
并发编程的本质就是在乱序执行的代码中创建小块的临界区，在临界区中程序线性执行，保证代码的执行结果符合预期。

# 互斥锁
sync.Mutex是互斥锁的实现，这是同步中最经典的模型，Mutex有两个方法Lock和Unlock，分别用于锁定和解锁一个锁，每个互斥锁只能被Lock一次，在一个已锁定的锁上执行Lock操作，会阻塞直到这个锁被Unlock。Mutex的声明如下：

```
type Mutex struct {
        // contains filtered or unexported fields
}

func (m *Mutex) Lock()

func (m *Mutex) Unlock()
```
对于前面那个程序，可以加入锁保证并发安全，同时推荐使用defer保证解锁

```
import (
      "fmt"
      "time"
      "sync"
)

func Add1(data *int, mu &sync.Mutex) {
     mu.Lock()
     defer mu.Unlock()
     tmp = *data
     *data = tmp + 1
}

func main(){
	  mu := sync.Mutex()
     data = 1
     go Add1(&data, &mu)
     go Add1(&data, &mu)
     time.Sleep(10)
     fmt.Print(data)
     
}
```
## 读写锁
实际上，单纯对数据的并发读取是不会带来数据不一致的，而这种情况下使用互斥锁会带来额外的等待开销，所以除了基础的互斥锁之外，sync包还提供了RWMutex，与Mutex不同在于，Mutex只能同时被一个线程锁定，而RWMutex可以多次读锁定，也就是可以进行并发读取，具体见[sync.RWMutex](https://golang.org/pkg/sync/#RWMutex)

# 信号量
sync.Cond是信号量的实现，这也是一个经典的同步模型，从功能的角度来看，你只能等待一个信号量或者向信号量发送一个信号，经典的应用场景就是生产者消费者模型，当生产者结束一个生产，则发起一个信号通知消费者；而消费者只需要等待这个信号。
## NewCond
Cond对象通过NewCond初始化，并且绑定到一个Locker。

```
type Cond struct {
        L Locker
}

func NewCond(l Locker) *Cond
```
## Signal&Broadcast
Signal和Broadcast用于唤醒一个信号量，区别在于Signal只会随机的唤醒一个线程，而Broadcast会唤醒所有在等待的线程。

```
func (c *Cond) Signal()

func (c *Cond) Broadcast()
```
## Wait
阻塞等待，当Signal或Broadcast被调用时唤醒，声明如下：

```
func (c *Cond) Wait()
```

# WaitGroup
WaitGroup的功能和信号量类似，不过信号量只等待单个信号的到来，WaitGroup等待一组任务的结束。
## Add
在一个WaitGroup上增加某个值，方法声明如下：

```
func (wg *WaitGroup) Add(delta int)
```

## Done
每次调用Done方法，会使WaitGroup的值减少 1

```
func (wg *WaitGroup) Done()
```

## Wait
阻塞等待，当WaitGroup的值为 0 时唤醒

```
func (wg *WaitGroup) Wait()
```

## 示例
下面用一个简单的例子对WaitGroup的使用进行说明，创建了一个WaitGroup并将其值增加3，然后并发的执行handle函数，最后调用Wait等待执行结束再退出主线程。
```
package main

import (
	"sync"
	"fmt"
)

func handle(wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Println("Done Once")
}

func main() {
	wg := sync.WaitGroup{}
	wg.Add(3)
	for i := 0; i < 3; i++ {
		go handle(&wg)
	}

	wg.Wait()
}
```

# Once
sync.Once保证某个函数有且仅有一次执行，只有一个方法Do，接受一个无参函数作为参数，当你调用Do时会执行这个函数，其方法声明如下：

```
func (o *Once) Do(f func())
```
下面的代码是对Once的简单使用，由于Once的使用，只会打印出一次“Do Once”，而不是10次

```
for i := 0; i < 10; i++ {
     sync.Once.Do(func () { fmt.Println("Do Once") }
}
```

# Cond
sync.Cond功能上和channal类似，但是sync.Cond缺乏传递变量，channal可以通过传递信号实现变量之间同步，但是sync.Cond也是一个比较方便的并发同步方法。在使用时上需要一些特别的faction设定。 在生产者消费者模型上比较合适。

>sync.Cond和其他一些sync包中的模块不同的是，sync.Cond需要自己实现流程方式，不能直接的开箱即用。

sync.Cond 有下列几种方法：
```
cond.L.Lock()和cond.L.Unlock()：也可以使用lock.Lock()和lock.Unlock()，完全一样，因为是指针转递

cond.Wait()：Unlock()->阻塞等待通知(即等待Signal()或Broadcast()的通知)->收到通知->Lock()

cond.Signal()：通知一个Wait()了的，若没有Wait()，也不会报错。Signal()通知的顺序是根据原来加入通知列表(Wait())的先入先出

cond.Broadcast(): 通知所有Wait()了的，若没有Wait()，也不会报错
```
声明方式通常为：
```
sync.NewCond({sync.Lock}.RLocker())
```
可能看得不太明白，下面之间展示常用的 生产者消费者模型：
```golang
package main

import (
	"bytes"
	"fmt"
	"io"
	"sync"
	"time"
)

type MyDataBucket struct {
	br     *bytes.Buffer
	gmutex *sync.RWMutex
	rcond  *sync.Cond //读操作需要用到的条件变量
}

func NewDataBucket() *MyDataBucket {
	buf := make([]byte, 0)
	db := &MyDataBucket{
		br:     bytes.NewBuffer(buf),
		gmutex: new(sync.RWMutex),
	}
	db.rcond = sync.NewCond(db.gmutex.RLocker())
	return db
}

func (db *MyDataBucket) Read(i int) {
	db.gmutex.RLock()
	defer db.gmutex.RUnlock()
	var data []byte
	var d byte
	var err error
	for {
		//读取一个字节
		if d, err = db.br.ReadByte(); err != nil {
			if err == io.EOF { // 输入流结束
				if string(data) != "" {
					fmt.Printf("reader-%d: %s\n", i, data)
				}
				db.rcond.Wait()
				data = data[:0] //data 清空
				continue
			}
		}
		data = append(data, d)
	}
}

//
func (db *MyDataBucket) Put(d []byte) (int, error) {
	db.gmutex.Lock()
	defer db.gmutex.Unlock()
	//写入一个数据块
	n, err := db.br.Write(d)
	//db.rcond.Broadcast() // 全通知
	db.rcond.Signal()    // 队列通知 FIFO
	return n, err
}

func main() {
	db := NewDataBucket()
	go db.Read(1)
	go db.Read(2)
	for i := 0; i < 10; i++ {
		go func(i int) {
			d := fmt.Sprintf("data-%d", i)
			db.Put([]byte(d))
		}(i)
		time.Sleep(100 * time.Millisecond)
	}
}
```