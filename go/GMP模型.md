# GMP模型

> Go 语言的调度器通过使用与 CPU 数量相等的线程减少线程频繁切换的内存开销，同时在每一个线程上执行额外开销更低的 Goroutine 来降低操作系统和硬件的负载。

1. GMP模型
   - `G`：即Goroutine 协程
   - `M`：内核级线程 G在操作系统中运行的实体
   - `P`：表示处理器 可以看作运行在本地线程M上的调度器  M与P是1:1的关系

![image-20220904165609064](D:\Program Files\电子书\go\md\图片\image-20220904165609064.png)

> 在runtime中 有三个全局变量 分别是allg allp allm 分别记录了空闲的gmp

```go
var(
	allp []*p
    allg []*g
    allm []*m
)
```

- 对于g来说 它既可以存储在p的本地运行列表 也可以存储在p的空闲列表 也可以存储在全局的g列表中
- 对于p来说 空闲的p即存储在全局的空闲p列表中 
- 对于m来说 空闲的m即存储在全局的空闲m列表中 自旋的线程m并不会保存在全局的空闲m列表中

## Work Stealing

-  当正活跃的线程中其p中已经没有可运行的g 且全局g列表为空时 那么就会执行`work stealing`机制



## G

1. **Goroutine是Golang语言调度器中待执行的任务 它在调度器中的地位与线程在操作系统中的地位相似 但是它相比线程对CPU来说占用了更少的资源 降低了上下文切换的开销(Goroutine上下文切换的开销 < 线程上下文切换的开销)**
2. 对于线程执行后的上下文切换 需要切换到内核态中保存堆栈信息 而Go中的协程执行后的上下文切换只需要在用户态就能实现 在Go程序中的运行时(runtime)帮我们执行上下文切换 即全局的调度器

3. Goroutine的结构

   ```go
   type g struct {
   	stack       stack
   	stackguard0 uintptr
       
       preempt       bool // 抢占信号
   	preemptStop   bool // 抢占时将状态修改成 `_Gpreempted`
   	preemptShrink bool // 在同步安全点收缩栈
       
       _panic       *_panic // 最内侧的 panic 结构体
   	_defer       *_defer // 最内侧的延迟函数结构体
       
       m              *m
   	sched          gobuf
   	atomicstatus   uint32
   	goid           int64
       ...
   }
   ```

   - `stack`描述当前Goroutine的栈内存范围[stack.lo, stack.hi]

   - `stackguard0` 可以用于调度器抢占式调度

   - `m` 描述当前G运行的实体M

   - **sched** 存储Goroutine的调度相关的数据

     - 这些字段会在调度器保存或恢复上下文的时候用到 其中的栈指针和程序计数器会用来存储或者恢复寄存器中的值，改变程序即将执行的代码。

       - `sp` 栈指针
       - `pc` 程序计数器
       - `g` 持有[`runtime.gobuf`](https://draveness.me/golang/tree/runtime.gobuf) 的 Goroutine
       - `ret` — 系统调用的返回值

       ```go
       type gobuf struct {
       	sp   uintptr
       	pc   uintptr
       	g    guintptr
       	ret  sys.Uintreg
       	...
       }
       ```

4. `runtime.g`中的`atomicstatus`保存了`g`当前的状态

   | 状态          | 描述                                                         |
   | ------------- | ------------------------------------------------------------ |
   | `_Gidle`      | 刚刚被分配并且还没有被初始化                                 |
   | `_Grunnable`  | 没有执行代码，没有栈的所有权，存储在运行队列中               |
   | `_Grunning`   | 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P  |
   | `_Gsyscall`   | 正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上 |
   | `_Gwaiting`   | 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上 |
   | `_Gdead`      | 没有被使用，没有执行代码，可能有分配的栈                     |
   | `_Gcopystack` | 栈正在被拷贝，没有执行代码，不在运行队列上                   |
   | `_Gpreempted` | 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒 |
   | `_Gscan`      | GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在      |



## M

1. M 即操作系统线程

2. Go调度器最多创建1000个线程 但是最多只有**GOMAXPROCS**个活跃线程能够正常执行 其他的线程可能都陷入了系统调用 或者在等待与P绑定

3. 默认情况下 运行时`runtime`会将`GOMAXPROCS`设置为当前机器CPU的核数 也可以通过在程序中使用`runtime`.`GOMAXPROCS`来改变最大的活跃线程数 例如一个4核的CPU会创建4个活跃的操作系统线程 在一般情况下我们都设置该参数为当前CPU的核数 这样**可以尽量减少操作系统线程级别的上下文切换和资源开销 所有的调度都只会发生在用户态 由Go的调度器触发**

4. M的结构

   ```go 
   type m struct {
   	g0   *g
   	curg *g
       p             puintptr
   	nextp         puintptr
   	oldp          puintptr
   	...
   }	
   ```

   - `g0`是持有调度栈的Goroutine 
   - `curg`是在当前M上运行的Goroutine
   - `p`是正在运行代码的处理器 (这里指的处理器是P)
   - `nextp`是暂存的处理器
   - `oldp`执行系统调用之前使用线程的处理器

   ![image-20220904171941108](D:\Program Files\电子书\go\md\图片\image-20220904171941108.png)

   ### g0

   1. `g0`是运行时中特殊的Goroutine 它参与运行时的调度过程 包括Goroutine的创建、大内存分配
   
      

## P (处理器)

1.  **调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率。**
2. 调度器在启动时就会创建 `GOMAXPROCS` 个处理器
3. `runtime.p`是处理器运行时的表示

```go
type p struct {
    // 当前与p绑定的线程
	m           muintptr

    // 表示处理器持有的运行队列，其中存储着待执行的 Goroutine 列表
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
    // 	线程下一个需要执行的 Goroutine。
	runnext guintptr
    
    // Available G's (status == Gdead)
    // Gdead 没有被使用，没有执行代码，可能有分配的栈
    // 空闲的G列表
	gFree struct {
		gList
		n int32
	}
	...
}
```

4. 处理器状态

| 状态        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `_Pidle`    | 处理器没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空 |
| `_Prunning` | 被线程 M 持有，并且正在执行用户代码或者调度器                |
| `_Psyscall` | 没有执行用户代码，当前线程陷入系统调用                       |
| `_Pgcstop`  | 被线程 M 持有，当前处理器由于垃圾回收被停止                  |
| `_Pdead`    | 当前处理器已经不被使用                                       |



## 调度器的启动过程

1. 运行时通过`runtime.schedinit()`初始化调度器
2. 在runtime.runtime2.go中初始化了一个全局的调度器 即`sched` 它就负责运行时的调度

```go
func schedinit() {
    // 得到当前执行的Goroutine的g
	_g_ := getg()
	...

    // 设置最大运行线程数
	sched.maxmcount = 10000
	...
    
    // 更新程序中处理器p的数量 完成相应数量处理器的启动 然后开始启动调度器
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
}
```



## 创建Goroutine

### 初始化g结构体

1. 使用`go`创建Goroutine时 编译器通过两个方法将该关键字转换成`runtime.newproc`函数调用

```go
func (s *state) call(n *Node, k callKind) *ssa.Value {
	if k == callDeferStack {
		...
	} else {
		switch {
		case k == callGo:
            // newproc
			call = s.newValue1A(ssa.OpStaticCall, types.TypeMem, newproc, s.mem())
		default:
		}
	}
	...
}
```

2. `newproc`的参数是待创建的Goroutine的参数大小 以及 表示该Goroutine的函数指针fn 它获取Goroutine以及调用方的`pc` 然后调用 [`runtime.newproc1`](https://draveness.me/golang/tree/runtime.newproc1) 函数获取新的 Goroutine 结构体、将其加入处理器的运行队列并在满足条件时调用 [`runtime.wakep`](https://draveness.me/golang/tree/runtime.wakep) 唤醒新的处理执行 Goroutine：

```go
func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg()
	pc := getcallerpc()
	systemstack(func() {
        // newproc1
		newg := newproc1(fn, argp, siz, gp, pc)

		_p_ := getg().m.p.ptr()
		runqput(_p_, newg, true)

		if mainStarted {
			wakep()
		}
	})
}
```

```go
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g {
    // 先得到要创建goroutine的创建方的goroutine
	_g_ := getg()
    // 待创建goroutine的参数大小
	siz := narg
	siz = (siz + 7) &^ 7

    // 得到创建方的goroutine附着的处理器P
	_p_ := _g_.m.p.ptr()
    // gfget即从p中的空闲G列表获取g 否则返回nil 
    // 由malg创建一个栈大小足够的新g结构体
	newg := gfget(_p_)
	if newg == nil {
		newg = malg(_StackMin)
		casgstatus(newg, _Gidle, _Gdead)
		allgadd(newg)
	}
	...
    totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize
	totalSize += -totalSize & (sys.SpAlign - 1)
	sp := newg.stack.hi - totalSize
	spArg := sp
	if narg > 0 {
        // 调用runtime.memmve函数将fn函数的所有参数拷贝到栈上
        // argp 和 narg 分别是参数的内存空间和大小
		memmove(unsafe.Pointer(spArg), argp, uintptr(narg))
	}
	...
```

- `runtime.gfget`通过两种不同的方式获取`rungime.g`

  1. 从 Goroutine 所在处理器的 `gFree` 列表或者调度器的 `sched.gFree` 列表中获取 [`runtime.g`](https://draveness.me/golang/tree/runtime.g)

     1. 当处理器的 Goroutine 列表为空时 会将调度器持有的空闲 Goroutine 转移到当前处理器上，直到 `gFree` 列表中的 Goroutine 数量达到 32

     2. 当处理器的 Goroutine 数量充足时 会从列表头部返回一个新的 Goroutine

        ```go
        func gfget(_p_ *p) *g {
        retry:
            // 如果p中没有g 而全局调度器sched中拥有 有栈帧的g 或者 没有栈帧的g
            // 则从全局调度器sched中得到空闲的g 直到p中的g达到32个
        	if _p_.gFree.empty() && (!sched.gFree.stack.empty() || !sched.gFree.noStack.empty()) {
        		for _p_.gFree.n < 32 {
        			gp := sched.gFree.stack.pop()
        			if gp == nil {
        				gp = sched.gFree.noStack.pop()
        				if gp == nil {
        					break
        				}
        			}
        			_p_.gFree.push(gp)
        		}
        		goto retry
        	}
            // 弹出一个g并返回
        	gp := _p_.gFree.pop()
        	if gp == nil {
        		return nil
        	}
        	return gp
        }
        ```

        

  2. 调用 [`runtime.malg`](https://draveness.me/golang/tree/runtime.malg) 生成一个新的 [`runtime.g`](https://draveness.me/golang/tree/runtime.g) 并将结构体追加到全局的 Goroutine 列表 `allgs` 中

  ```go
  // runtime.proc.go
  var allgs    []*g
  ```

  ![image-20220904183303236](D:\Program Files\电子书\go\md\图片\image-20220904183303236.png)

### 运行队列

1. [`runtime.runqput`](https://draveness.me/golang/tree/runtime.runqput) 会将 Goroutine 放到运行队列上

   - 当next为true 将刚创建的g设到处理器的`runnext`作为处理器下一个运行的任务
   - 当 `next` 为 `false` 并且本地运行队列还有剩余空间时，将 Goroutine 加入处理器持有的本地运行队列
   - 当处理器的本地运行队列已经没有剩余空间时就会把本地队列中的一部分 Goroutine 和待加入的 Goroutine 通过 [`runtime.runqputslow`](https://draveness.me/golang/tree/runtime.runqputslow) 添加到调度器持有的全局运行队列上
   - 处理器p中是包含运行g队列和空闲g队列的

   ```go
   // gp 刚创建的g
   func runqput(_p_ *p, gp *g, next bool) {
   	if next {
   	retryNext:
   		oldnext := _p_.runnext
   		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
   			goto retryNext
   		}
   		if oldnext == 0 {
   			return
   		}
   		gp = oldnext.ptr()
   	}
   retry:
   	h := atomic.LoadAcq(&_p_.runqhead)
   	t := _p_.runqtail
       // 本地队列还有剩余空间
   	if t-h < uint32(len(_p_.runq)) {
   		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
   		atomic.StoreRel(&_p_.runqtail, t+1)
   		return
   	}
       // 本地队列中的一部分 Goroutine 和待加入的 Goroutine添加到调度器sched持有的全局运行队列上
   	if runqputslow(_p_, gp, h, t) {
   		return
   	}
   	goto retry
   }
   ```

2. **处理器本地的运行队列是一个使用数组构成的环形链表，它最多可以存储 256 个待执行任务。**

   ![image-20220904184554592](D:\Program Files\电子书\go\md\图片\image-20220904184554592.png)

## 调度循环

1. 调度器启动之后，Go 语言运行时会调用 [`runtime.mstart`](https://draveness.me/golang/tree/runtime.mstart) 以及 [`runtime.mstart1`](https://draveness.me/golang/tree/runtime.mstart1)，前者会初始化 g0 的 `stackguard0` 和 `stackguard1` 字段，后者会初始化线程并调用 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 进入调度循环：

```go
func schedule() {
	_g_ := getg()

top:
	pp := _g_.m.p.ptr()
	pp.preempt = false

	if sched.gcwaiting != 0 {
		gcstopm()
		goto top
	}
	if pp.runSafePointFn != 0 {
		runSafePointFn()
	}

	// Sanity check: if we are spinning, the run queue should be empty.
	// Check this before calling checkTimers, as that might call
	// goready to put a ready goroutine on the local run queue.
	if _g_.m.spinning && (pp.runnext != 0 || pp.runqhead != pp.runqtail) {
		throw("schedule: spinning with local work")
	}

	checkTimers(pp, 0)

	var gp *g
	var inheritTime bool

	// Normal goroutines will check for need to wakeP in ready,
	// but GCworkers and tracereaders will not, so the check must
	// be done here instead.
	tryWakeP := false
	if trace.enabled || trace.shutdown {
		gp = traceReader()
		if gp != nil {
			casgstatus(gp, _Gwaiting, _Grunnable)
			traceGoUnpark(gp, 0)
			tryWakeP = true
		}
	}
	if gp == nil && gcBlackenEnabled != 0 {
		gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
		if gp != nil {
			tryWakeP = true
		}
	}
	if gp == nil {
		// Check the global runnable queue once in a while to ensure fairness.
		// Otherwise two goroutines can completely occupy the local runqueue
		// by constantly respawning each other.
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
		gp, inheritTime = runqget(_g_.m.p.ptr())
		// We can see gp != nil here even if the M is spinning,
		// if checkTimers added a local goroutine via goready.
	}
	if gp == nil {
		gp, inheritTime = findrunnable() // blocks until work is available
	}

	// This thread is going to run a goroutine and is not spinning anymore,
	// so if it was marked as spinning we need to reset it now and potentially
	// start a new spinning M.
	if _g_.m.spinning {
		resetspinning()
	}

	if sched.disable.user && !schedEnabled(gp) {
		// Scheduling of this goroutine is disabled. Put it on
		// the list of pending runnable goroutines for when we
		// re-enable user scheduling and look again.
		lock(&sched.lock)
		if schedEnabled(gp) {
			// Something re-enabled scheduling while we
			// were acquiring the lock.
			unlock(&sched.lock)
		} else {
			sched.disable.runnable.pushBack(gp)
			sched.disable.n++
			unlock(&sched.lock)
			goto top
		}
	}

	// If about to schedule a not-normal goroutine (a GCworker or tracereader),
	// wake a P if there is one.
	if tryWakeP {
		wakep()
	}
	if gp.lockedm != 0 {
		// Hands off own p to the locked m,
		// then blocks waiting for a new p.
		startlockedm(gp)
		goto top
	}

	execute(gp, inheritTime)
}
```

- 为了保证公平，当全局运行队列中有待执行的 Goroutine 时，通过 `schedtick` 保证有一定几率会从全局的运行队列中查找对应的 Goroutine

- 从处理器本地的运行队列中查找待执行的 Goroutine

- 如果前两种方法都没有找到 Goroutine，会通过 [`runtime.findrunnable`](https://draveness.me/golang/tree/runtime.findrunnable) 进行阻塞地查找 Goroutine

  - [`runtime.findrunnable`](https://draveness.me/golang/tree/runtime.findrunnable) 的实现非常复杂，这个 300 多行的函数通过以下的过程获取可运行的 Goroutine：

    1. 从本地运行队列、全局运行队列中查找

    2. 从网络轮询器中查找是否有 Goroutine 等待运行

    3. 通过 [`runtime.runqsteal`](https://draveness.me/golang/tree/runtime.runqsteal) 尝试从其他随机的处理器中窃取待运行的 Goroutine，该函数还可能窃取处理器的计时器

       

2. 接下来由 [`runtime.execute`](https://draveness.me/golang/tree/runtime.execute) 执行获取的 Goroutine，做好准备工作后，它会通过 [`runtime.gogo`](https://draveness.me/golang/tree/runtime.gogo) 将 Goroutine 调度到当前线程上

   ```go
   func execute(gp *g, inheritTime bool) {
   	_g_ := getg()
   
   	_g_.m.curg = gp
   	gp.m = _g_.m
   	casgstatus(gp, _Grunnable, _Grunning)
   	gp.waitsince = 0
   	gp.preempt = false
   	gp.stackguard0 = gp.stack.lo + _StackGuard
   	if !inheritTime {
   		_g_.m.p.ptr().schedtick++
   	}
   
   	gogo(&gp.sched)
   }
   ```

   

3. 