# sync.Pool

1. Pool：结构体
   
   ```go
   # 结构体字段
   type Pool struct {
   	noCopy noCopy
   
   	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
   	localSize uintptr        // size of the local array
   
   	victim     unsafe.Pointer // local from previous cycle
   	victimSize uintptr        // size of victims array
   
   	// New optionally specifies a function to generate
   	// a value when Get would otherwise return nil.
   	// It may not be changed concurrently with calls to Get.
   	New func() any
   }
   
   # Pool方法
   func (p *Pool) Get() interface{}
   func (p *Pool) Put(x interface{})
   ```
   
   1. 通过New字段来新建池中需要的某种类型的具体值 可以放字符串 整型数等
   
   2. 通过Get方法可以从Pool池中获取到我们之前在New里面定义类型的数据
   
   3. 当我们用完以后 可以通过Pool.Put将该item放回池中 或者可以存放别的同类型数据进去
   
   4. 池的目的是为了复用已经使用过的对象 来达到优化内存和回收的目的 池创建时会初始化并提供一些相同对象 当程序需要很多池的同类型数据且池当前对象不够时 会通过New新产生一些 
   
   5. ```go
      Pool's purpose is to cache allocated but unused items for later reuse, 
      relieving pressure on the garbage collector. 
      That is, it makes it easy to build efficient, thread-safe free lists
      ```
   
   6. ```go
      # 例子
      func main(){
          pool := sync.Pool{
      		New: func() interface{} {
      			return 5
      		},
      	}
      	a := pool.Get().(int)
      	fmt.Println(a)
      	pool.Put(3) //Put之后New中返回值重新改变
      	//runtime.GC()
      	b := pool.Get().(int)
      	fmt.Println(b)
      }
      
      # 输出
      // 5
      // 3
      ```
   
   7. Get即从池中取出任意的一个item
   
   8. Put即将item返回池中 并可以修改其返回值
