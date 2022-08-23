# Channel

> Kavas  https://files.speakerdeck.com/presentations/10ac0b1d76a6463aa98ad6a9dec917a7/GopherCon_v10.0.pdf
>
> 讲解Channel源码 https://codeburst.io/diving-deep-into-the-golang-channels-549fd4ed21a8
>
> Do not communicate by sharing memory; instead, share memory by communicating.

1. `Channel`的底层数据结构

   ```go
   // hchan struct
   type hchan struct {
       // 当前管道中元素的个数 当qcount==dataqsiz时说明管道满了
   	qcount   uint           // total data in the queue
       // 环形队列的容量
   	dataqsiz uint           // size of the circular queue
       // buf是指向有dataqsiz容量的数组的一个指针
   	buf      unsafe.Pointer // points to an array of dataqsiz elements
       // 数组中元素类型占用的字节数
   	elemsize uint16
       // 管道是否关闭 
       // 0：未关闭 
       // 1：关闭
   	closed   uint32
       // 指向数组中元素的类型描述信息(类型元数据)
   	elemtype *_type 
       // 当前环形队列中发送goroutine写入数据的位置(数组索引)
   	sendx    uint   
       // 当前环形队列中接收goroutine接收数据的位置(数组索引)
   	recvx    uint   
       // 正在阻塞的接收goroutine队列
   	recvq    waitq  // list of recv waiters
       // 正在阻塞的发送goroutine队列
   	sendq    waitq  // list of send waiters
   
   	// lock protects all fields in hchan, as well as several
   	// fields in sudogs blocked on this channel.
   	//
   	// Do not change another G's status while holding this lock
   	// (in particular, do not ready a G), as this can deadlock
   	// with stack shrinking.
   	lock mutex
   }
   ```

   ![img](https://miro.medium.com/max/443/1*zjKfZFnvkZ9eZrBw4DYXfg.png)

2. `recvq`和`sendq`中指向的是一个`waitq`结构  `waitq`中有两个指针 分别指向接收队列或发送队列的头部和尾部 

   `sudog`是存储`接收/发送Goroutine`等数据的结构

   ```go
   type waitq struct {
   	first *sudog
   	last  *sudog
   }
   
   type sudog struct {
   	// The following fields are protected by the hchan.lock of the
   	// channel this sudog is blocking on. shrinkstack depends on
   	// this for sudogs involved in channel ops.
   
       // 当前运行的Goroutine
   	g *g
   
   	next *sudog
   	prev *sudog
   	elem unsafe.Pointer // data element (may point to stack)
   
   	// The following fields are never accessed concurrently.
   	// For channels, waitlink is only accessed by g.
   	// For semaphores, all fields (including the ones above)
   	// are only accessed when holding a semaRoot lock.
   
   	acquiretime int64
   	releasetime int64
   	ticket      uint32
   
   	// isSelect indicates g is participating in a select, so
   	// g.selectDone must be CAS'd to win the wake-up race.
   	isSelect bool
   
   	// success indicates whether communication over channel c
   	// succeeded. It is true if the goroutine was awoken because a
   	// value was delivered over channel c, and false if awoken
   	// because c was closed.
   	success bool
   
   	parent   *sudog // semaRoot binary tree
   	waitlink *sudog // g.waiting list or semaRoot
   	waittail *sudog // semaRoot
   	c        *hchan // channel
   }
   ```

![img](https://miro.medium.com/max/583/1*fiFgoUaJ8nV-SQtEjRv2-w.jpeg)

3. `Channel`的`Send`发送数据操作

   **发送在管道上的所有数据都只是值的副本 修改原先的值并不会改变管道中已经存储的数据**

   ```go
   func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
       // 1. 如果管道为空 即buf上没有数据 且recvq为空 所有字段都为默认值
       if c == nil {
   		if !block {
   			return false
   		}
           // 将当前gotoutine置为阻塞 将当前send操作延迟到g被唤醒为止
   		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
   		throw("unreachable")
   	}
       
       // 2. 在已经关闭的管道上发送数据 会直接panic
       if c.closed != 0 {
   		unlock(&c.lock)
   		panic(plainError("send on closed channel"))
   	}
   
       // 3. 当前管道上recvq有存储等待接收数据的goroutine 直接取出该sudog结构 
       // 3. 并直接将ep指向的真实数据直接发送给sudog中的elem字段
       if sg := c.recvq.dequeue(); sg != nil {
   		// Found a waiting receiver. We pass the value we want to send
   		// directly to the receiver, bypassing the channel buffer (if any).
   		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
   		return true
   	}	
       
       // sg == nil recvq上没有数据
       // 4. 将ep指向的数据发送到缓存上
       if c.qcount < c.dataqsiz {
           // Space is available in the channel buffer. Enqueue the element to send.
           qp := chanbuf(c, c.sendx)
           if raceenabled {
               racenotify(c, c.sendx, nil)
           }
           // 发送数据到buf中
           typedmemmove(c.elemtype, qp, ep)
           c.sendx++
           if c.sendx == c.dataqsiz {
               c.sendx = 0
           }
           c.qcount++
           unlock(&c.lock)
           return true
       }
       
       // buf中容量已满
       // 5. 将当前g阻塞 新建sudog结构 将g和ep赋值到sudog结构中并入队sendq 等待被唤醒
       // Block on the channel. Some receiver will complete our operation for us.
   	gp := getg()
   	mysg := acquireSudog()
   	mysg.releasetime = 0
   	if t0 != 0 {
   		mysg.releasetime = -1
   	}
   	// No stack splits between assigning elem and enqueuing mysg
   	// on gp.waiting where copystack can find it.
   	mysg.elem = ep
   	mysg.waitlink = nil
   	mysg.g = gp
   	mysg.isSelect = false
   	mysg.c = c
   	gp.waiting = mysg
   	gp.param = nil
   	c.sendq.enqueue(mysg)
       
       ···
   }
   
   // 将goroutine设置为阻塞
   gopark()
   // 将goroutine设置为runable
   goready()
   ```

   ## Send operation Summary

   1. **lock** the entire channel structure.

   2. determines writes. Try `recvq` to take a waiting goroutine from the wait queue, then hand the element to be written directly to the goroutine.

   3. If recvq is Empty, Determine whether the buffer is available. If available, ***\*copy\**** (`typedmemmove` copies a value of type t to dst from src.`) the data from current goroutine to the buffer.
      *_typedmemmove_* internally uses *memmove — memmove() is used to copy a block of memory from a location to another.*

   4. If the buffer is full then the element to be written is saved in the structure of the currently executing goroutine and the current goroutine is enqueued at **sendq** and suspended, from runtime.(对于没有缓存的管道来说 发送者是直接将待发送数据发送到接收者的elem上)

      如这个例子 A读取管道c中的数据 但此时没有缓存也没有发送者发送数据 此时A会阻塞 加入c的recvq中 并且t也会加入该A所在的`sudog`结构中的`elem` 

      此时如果B发送了一个数据 那么B发送的数据将直接发送到`recvq`中A的sudog结构中的elem 即`t`

      ```go
      func goroutineA(c chan int){
          // 从c中读取数据
          t := <- c
      }
      func main(){
          c := make(chan int)
          go gotoutineA(c)
          
          // 防止main协程结束
          for{ 
          }
      }
      
      // 发送数据版
      func goroutineB(c chan int){
          // 发送数据
          t := 6
          c <- t
      }
      func main(){
          c := make(chan int)
          go goroutineA(c)
          go goroutineB(c)
          // 防止main协程结束
          for{   
          }
      }
      ```

      

4. `Channel`的`Recv`接收数据操作与发送数据操作原理相似