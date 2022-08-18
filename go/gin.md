# Gin源码学习

```go
# 创建路由引擎
// r 是 *Engine结构体类型 
// Engine嵌套了RouterGroup结构体 因此可以调用Group方法
r := gin.Default()

# 客户端访问的路径：/p1
# 客户端访问/p1的方法：GET
# 访问/p1所进行的回调函数：可以自己编写函数所需要进行的操作
r.GET("/p1", func (c *gin.Context){} )

// 默认路由可以分多个路由组
api := r.Group("/api/v1")

// HandleFunc ：func (*gin.Context)类型
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup

# 调用Use函数在路由方法前可以使用中间件
// IRouters是一个拥有所有路由方法的接口
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes
```

```go
# Gin中间件加载过程

// middleware是我们自己编写的函数 类型 : HandleFunc-> func (c *gin.Context)
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
    
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}

// 使用方法: r.Use(middleware1, middleware2).GET("/p2", get_handler) 
// 先进行中间件加载middleware1、middleware2 然后在加载路由的处理函数 get_handler

# 具体实现
type RouterGroup struct {
	Handlers HandlersChain
	basePath string
	engine   *Engine
	root     bool
}
type HandlersChain []HandlerFunc
type HandlerFunc func ( *gin.Context)
// Use函数: group.Handlers = append(group.Handlers, middleware...)
// 由此得知Handlers是一个存储func ( *gin.Context)的切片 即存储中间件和路由的所有回调函数
// 所以Use函数先将中间件的回调函数存储到Handlers这个切片中 等待调用

// 查看路由的具体实现过程
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
    // 调用了handle方法
    // http.MethodGet：因为路由调用了GET方法
    // relativePath：GET的相对路径 若该路由是由前面的路由创建出来的 则需要进行拼接来访问绝对路径
    // handlers：路由的回调函数 会追加到RoterGroup的Handlers中
    // 不同的路由或路由组的Handlers是不同的
	return group.handle(http.MethodGet, relativePath, handlers)
}

func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
    
    // 将定义的路由相对路径拼接为完整的绝对路径
    // 因为一个完整的路径往往和前面定义的路由组路径有关系，因此需要一系列拼接
	absolutePath := group.calculateAbsolutePath(relativePath)
   
    // 针对完整路由路径将关联的回调函数全部组合出来
	handlers = group.combineHandlers(handlers)
    
    // 将请求方式（GET、POST等）结合完整路径作为key，处理函数作为 value
    // 以 key => value 的形式注册，value 可以是很多个回调函数
	group.engine.addRoute(httpMethod, absolutePath, handlers)
	return group.returnObj()
}

```

```go
# combineHandlers函数
func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
    
    // finalSize 表示 中间件回调函数的数量 + 具体路由的回调函数数量的总和
	finalSize := len(group.Handlers) + len(handlers)
    
    // 如果 finalSize >= abortIndex 就会发生panic，
    // abortIndex 的定义值： math.MaxInt8 / 2 
    // 在go语言中，官方定义： MaxInt8   = 1<<7 - 1， 表示 1*(2^7)-1,最终值：127
    // 那么  abortIndex = 127/2 取整 = 63
    // 至此我们终于知道, gin 的中间件函数数量 + 路由回调函数的数量总和最大允许 63 个
    // 回调函数的存储索引是从0开始的，最大索引为：62
	if finalSize >= int(abortIndex) {
		panic("too many handlers")
	}
    
    // 以上条件检查全部通过后，将中间件回调函数和路由回调函数全部合并在一起存储
    // HandlersChain 本质就是 [] func(*Context)
	mergedHandlers := make(HandlersChain, finalSize)    
    // group.Handlers 是中间件函数，他在 mergedHandlers 中存储的顺序靠前，也就是索引比较小
	copy(mergedHandlers, group.Handlers)
    // handlers 是路由回调函数，他的存储位置比中间函数靠后
	copy(mergedHandlers[len(group.Handlers):], handlers)
    
    // 最终返回中间件函数+路由函数组合在一起的全部回调函数
	return mergedHandlers
}
```



![image-20220803145921131](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220803145921131.png)



```go
# assert1
func assert1(guard bool, text string) {
    // guard需为真才不会panic
	if !guard { 
		panic(text)
	}
}

# trees.get(method)
type methodTree struct {
	method string
	root   *node
}

type methodTrees []methodTree

func (trees methodTrees) get(method string) *node {
	for _, tree := range trees {
		if tree.method == method {
			return tree.root
		}
	}
	return nil
}

# addRoute
func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
    
	assert1(path[0] == '/', "path must begin with '/'")
	assert1(method != "", "HTTP method can not be empty")
	assert1(len(handlers) > 0, "there must be at least one handler")

	debugPrintRoute(method, path, handlers)
	
    // 得到method对应的根结点
	root := engine.trees.get(method)
	if root == nil {
        // 为空则首次创建root根结点
		root = new(node)
        // 设置路径为 /
		root.fullPath = "/"
        // 设置tress中对应的方法的根节点 可能会有4个根结点
		engine.trees = append(engine.trees, methodTree{method: method, root: root})
	}
    
    // 在这里，gin 将我们定义的完整路径和回调函数进行了注册
    // 为请求的方法进行回调函数的添加
	root.addRoute(path, handlers)

	// Update maxParams
	if paramsCount := countParams(path); paramsCount > engine.maxParams {
		engine.maxParams = paramsCount
	}
}

//  node 的结构体定义，发现就是自己嵌套自己的一个结构体
// handlers 成员的数据类型（HandlersChain）本质就是 []func(*gin.Context)
// 路由的存储模型是一个树形结构，每个节点都有自己路由路径以及回调函数 handlers
type node struct {
	path      string
	indices   string
	wildChild bool
	nType     nodeType
	priority  uint32
	children  []*node 		// child nodes, at most 1 :param style node at the end of the array
	handlers  HandlersChain // 该node结点对应的执行函数 （根据该HandlersChain进行回调函数的调用
	fullPath  string
}

// 6.root.addRoute(path, handlers) 函数在按照 键 => 值 进行注册路由与回调函数时调用了很多其他函数
func (n *node) addRoute(path string, handlers HandlersChain) {
    
        // 省略其他无关代码
		n.insertChild(path, fullPath, handlers)
    
    	// 省略很多其他的代码
}

// 7. n.insertChild
func (n *node) insertChild(path string, fullPath string, handlers HandlersChain) {
    
    //省略其他无关代码
    
    // 这里的n即为方法对应的根结点
    // child会append到node的children中
    child := &node{
        priority: 1,
        fullPath: fullPath,  // 完整的路由路径
    }
    
    // 最终所有的回调函数
    n.handlers = handlers
}
```

![image-20220803160213120](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220803160213120.png)

### Gin路由背后的所有回调函数什么时候会执行

1. 

   ```go
   r := gin.Default()
   
   r.Run()
   
   # Run
   func (engine *Engine) Run(addr ...string) (err error) {
      	// 省略部分代码
        err = http.ListenAndServe(address, engine)
       return
   }
   
   # ListenAndServe
   # addr:是IP:Port格式
   // !!! 只要实现了该接口 传递实现的原型
   // !!! Go官方的net库会回调该函数
   type Handler interface {
   	ServeHTTP(ResponseWriter, *Request)
   }
   func ListenAndServe(addr string, handler Handler) error {
   	server := &Server{Addr: addr, Handler: handler}
   	return server.ListenAndServe()
   }
   
   
   
   func (engine *Engine) ServeHTTP (w http.ResponseWriter, req *http.Request) {
   	// 初始化gin的Context上下文成员参数
       // 省略部分代码
       engine.handleHTTPRequest(c)
   }
   
   func (engine *Engine) handleHTTPRequest(c *Context) {
       	// 这里根据客户端实际请求的路径、参数，大小写不敏感模式去寻找已经注册的路由表中对应的回调函数
       	value := root.getValue(rPath, c.params, unescape)
   		if value.params != nil {
   			c.Params = *value.params
   		}
   		if value.handlers != nil {
              
                //value.handlers  路由键对应的全部回调函数
   			c.handlers = value.handlers
               
                // 路由全路径
   			c.fullPath = value.fullPath
          
               // 最核心的东西，所有回调函数要开始执行了
               // -----------------------------------
   			c.Next()
               // -----------------------------------
   			c.writermem.WriteHeaderNow()
   			return
   		}
   
       
       // 省略其他代码....
   }
   
   // 根据Context中的路由全路径开始执行回调函数
   func (c *Context) Next() {
   	c.index++
       
       // 开发者在任何一个回调函数只要调用了 Abort 就会随时终止后面的回调函数执行
       // 具体参见后面第 7 条，以及前文分析的最大回调函数总数量为：63 个
   	for c.index < int8(len(c.handlers)) {
           
           // 这里按照回调函数最开始的注册顺序，去执行.
   		c.handlers[c.index](c)
   		c.index++
   	}
   }
   // 如果开发者在任何一个回调函数调用了本函数，那么 index 值瞬间就被设置为 63
   func (c *Context) Abort() {
       // abortIndex 为一个常量值：63
   	c.index = abortIndex
   }
   ```
   
   ![image-20220803184442220](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220803184442220.png)