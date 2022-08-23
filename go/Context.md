# Context

1. `Context`是一个接口 对应有4种空方法

   ```go
   type Context interface{
       Value(key any) any // any : interface alias
   	Err() error
       Done() <-chan struct{}
       Deadline() (deadline time.Time, ok bool)
   }
   ```

   

2. 新建一个`Context`：`ctx := context.Background()`

   ```go
   type emptyCtx int // 主要用于新建立一个独立的树结构
   
   var (
   	background = new(emptyCtx)
   	todo       = new(emptyCtx)
   )
   
   func TODO() Context {
   	return todo
   }
   
   func Background() Context {
   	return background
   }
   
   // context内置创建了两个emptyCtx : background / todo 
   // 对应的函数将进行返回
   ```

   <img src="C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220823153051956.png" alt="image-20220823153051956" style="zoom: 67%;" />

   

3. Context的4种具体类型

   - `emptyCtx`  ： `TODO() / Background()创建`

     -  一个空的 ctx，一般用于做根节点

   - `cancelCtx` ：`WithCancel()创建` 

     -  核心，用来处理取消相关的操作

   - `timerCtx`   ：`WithDeadLine()创建` 

     - 用来处理超时相关操作

   - `valueCtx`  ：`WithValue()创建`  

     - 附加值的实现方法

     

4. `valueCtx` 

   ```go
   type valueCtx struct {
       // 存储父context
   	Context
   	key, val any
   }
   ```

   

   ```go
   // 设置key, value值
   // 返回valueCtx的Context
   func WithValue(parent Context, key, val interface{}) Context {
      if key == nil {
         panic("nil key")
      }
      if !reflectlite.TypeOf(key).Comparable() {
         panic("key is not comparable")
      }
      // 在当前节点下生成新的子节点
      return &valueCtx{parent, key, val}
   }
   
   // 根据key读取value
   func (c *valueCtx) Value(key interface{}) interface{} {
      // 在此Context中得到值
       if c.key == key {
         return c.val
      }
       // c.Context是c的父context 从它一直往上递归key直到得到值或到根节点nil返回
      return c.Context.Value(key)
   }
   ```

   <img src="C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220823153545103.png" alt="image-20220823153545103" style="zoom: 67%;" />

   

5. `cancelCtx`

   结构

   ```go
   type cancelCtx struct {
      Context
   
      mu       sync.Mutex            // protects following fields
      done     atomic.Value        // created lazily, closed by first cancel call
      children map[canceler]struct{} // set to nil by the first cancel call
      err      error                 // set to non-nil by the first cancel call
   }
   
   type canceler interface {
      cancel(removeFromParent bool, err error)
      Done() <-chan struct{}
   }
   ```

   - **done : 用于判断该Context是否完成(被Cancel/超时)**
   - **children : 存放该Context的子Context**
   - **err ：取消时的错误，超时或主动取消 未取消时为nil**

   对外方法

   ```go
   // 创建一个cancelCtx
   func withCancel(parent Context){
       // c.Context = parent
      c := newCancelCtx(parent)
      // 将父节点的取消函数和子节点关联，做到父节点取消，子节点也跟着取消
      propagateCancel(parent, &c)
      // 返回当前节点和主动取消函数（调用会将自身从父节点移除，并返回一个已取消错误）
      return &c, func() { c.cancel(true, Canceled) }
   }
   
   ```

   **绑定父子结点的取消关系**

   ① 当祖先继承链里没有 cancelCtx 或 timerCtx 等实现时，Done()方法总是返回 nil，可以作为前置判断

   ② parentCancelCtx 取的是可以取消的最近祖先节点

   ```go
   // propagateCancel(parent, &c)
   // propagateCancel arranges for child to be canceled when parent is.
   func propagateCancel(parent Context, child canceler) {
      done := parent.Done()
      if done == nil {
       // 若当前节点追溯到根没有cancelCtx或者timerCtx的话，表示当前节点的祖先没有可以取消的结构，后面的父子绑定的操作就可以不用做了，可参考下图
         return // parent is never canceled
      }
   
      select {
      case <-done:
         // 父节点已取消就直接取消子节点，无需移除是因为父子关系还没加到parent.children
         // parent is already canceled
         child.cancel(false, parent.Err())
         return
      default:
      }
   
     // 获取最近的可取消的祖先
      if p, ok := parentCancelCtx(parent); ok {
         p.mu.Lock()
         if p.err != nil {
       // 和前面一样，如果祖先节点已经取消过了，后面就没必要绑定，直接取消就好
            // parent has already been canceled
            child.cancel(false, p.err)
         } else {
        // 绑定父子关系
            if p.children == nil {
               p.children = make(map[canceler]struct{})
            }
            p.children[child] = struct{}{}
         }
         p.mu.Unlock()
      } else {
       // 当ctx是开发者自定义的并继承context.Context接口会进入这个分支，另起一个协程来监听取消动作，因为开发者自定义的习惯可能和源码中用c.done和c.err的判断方式有所不同
         atomic.AddInt32(&goroutines, +1)
         go func() {
            select {
            case <-parent.Done():
               child.cancel(false, parent.Err())
            case <-child.Done():
            }
         }()
      }
   }
   
   // 为什么能够找到父context往上的CancelCtx? 就在这个函数中
   func parentCancelCtx(parent Context) (*cancelCtx, bool) {
   	done := parent.Done()
   	if done == closedchan || done == nil {
   		return nil, false
   	}
       // 在这里寻找祖先继承链前的CancelCtx
       // Value函数在cancelCtx中实现 若往上寻找的结点是cancelCtx 
       // 那么这个结点的Value中key与cancelCtxKey比较 正确的话则返回存放动态类型c的结构 
       // 然后断言为*cancelCtx
   	p, ok := parent.Value(&cancelCtxKey).(*cancelCtx)
   	if !ok {
   		return nil, false
   	}
   	pdone, _ := p.done.Load().(chan struct{})
   	if pdone != done {
   		return nil, false
   	}
   	return p, true
   }
   
   func (c *cancelCtx) Value(key any) any {
   	if key == &cancelCtxKey {
   		return c
   	}
   	return value(c.Context, key)
   }
   ```

   <img src="C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220823162516497.png" alt="image-20220823162516497" style="zoom:67%;" />

   

   **cancel函数实现**

   ```go
   // cancel closes c.done, cancels each of c's children, and, if
   // removeFromParent is true, removes c from its parent's children.
   func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    	// 取消无论是通过父节点还是自身主动取消，err都不为空
       // 因此若err为nil则表示错误
      if err == nil {
         panic("context: internal error: missing cancel error")
      }
      c.mu.Lock()
      if c.err != nil {
       // c.err 不为空表示已经被取消过，比如父节点取消时子节点可能已经主动调用过取消函数
         c.mu.Unlock()
         return // already canceled
      }
      c.err = err
       
      if c.done == nil {
       // closedchan 是一个已经关闭的channel，要特殊处理是因为c.done是懒加载的方式。只有调用c.Done()时才会实际创建
         c.done = closedchan
      } else {
         close(c.done)
      }
      // 递归取消子节点
      for child := range c.children {
         // NOTE: acquiring the child's lock while holding parent's lock.
         child.cancel(false, err)
      }
       // 因此当子结点全部取消后 将当前结点的children置为nil
      c.children = nil
      c.mu.Unlock()
   
    // 从父节点中移除当前节点
    // removeFromParent为false时则不会从父结点移除 因为当前父结点的锁已被获取
    // 这时再获取锁会造成冲突 
      if removeFromParent {
         removeChild(c.Context, c)
      }
   }
   
   // removeChild removes a context from its parent.
   func removeChild(parent Context, child canceler) {
   	p, ok := parentCancelCtx(parent)
   	if !ok {
   		return
   	}
   	p.mu.Lock()
   	if p.children != nil {
   		delete(p.children, child)
   	}
   	p.mu.Unlock()
   }
   
   ```

   Done函数

   ```go
   // 若父context不为cancelCtx 那么会往上追溯 直到找到类型为cancelCtx的context才返回 否则返回nil
   ```

   

6. 