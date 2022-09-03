# Gin中的路由规则匹配

## 什么是基数树

1. 基数树 ：也称压缩前缀树
2. 原理：**树中每个结点都是子节点的最长公共前缀**

![image-20220904000203756](D:\Program Files\电子书\go\md\图片\image-20220904000203756.png)

3. **基数树在Go路由中的应用**

   - 对于第二步中`path`为`abc`的`node`结点来说 在执行`route.GET("abd")` 该结点将会**分裂** 因为`abd`与`abc`有公共前缀`a` 
   - 具体怎么**分裂**呢 在后面的`func (engine *Engine) addRoute`中将会讲解
   - 其实代码中具体的意思就是新创建一个`child`结点 把`abc`这个node结点相当于复制给了`child` 只不过其path修改为公共前缀之后的字符串 以及`priority`自减1 
   - 然后当前的这个node结点的`handlers`置为nil 因为不是完整的路由路径 所以没有对应的`handlers` 以及将`path`和`fullpath`进行修改 增加`child`结点以及即将添加进来的node结点(这里指的是`path`为`d`的结点)

   ![image-20220904001136886](D:\Program Files\电子书\go\md\图片\image-20220904001136886.png)

## Gin路由代码详解

1.  Gin中`GET` \ `POST` \ `DELETE` \ `PUT` 都会生成对应的结点结构

   ```go
   type node struct {
       // 当前node存储的路径 
   	path      string
       // 所有子节点path的第一个字符组成的字符串
   	indices   string
       // 是否为参数节点 如果是->wildChild=true
   	wildChild bool
       // 当前结点的类型
       // static 静态结点 例如/id /user之类
       // root	  四个方法对应的根节点 
       // param  参数结点	例如/user/:id
       // catchAll 通用匹配，匹配任意参数(*user)
   	nType     nodeType
       // 即当前node的子结点(子子结点...)的个数总和
   	priority  uint32
       // 子节点
   	children  []*node // child nodes, at most 1 :param style node at the end of the array
   	// 当前node对应的handler
       handlers  HandlersChain
       // 当前node的绝对路径 = 兄弟节点的公共前缀 + node.path
   	fullPath  string
   }
   
   const (
   	static nodeType = iota // default
   	root
   	param
   	catchAll
   )
   ```

   

2. 开始注册路由

   ```go
   r := gin.Default()
   // r's type = engine 
   // engine中内嵌了RouterGroup结构体 因此可以调用GET/POST/DELETE/PUT方法
   
   // 这一句可以忽略 只是为了方便后面addRoute时深入了解 后面会提及为什么忽略
   r.GET("/ss")
   
   // foo是访问到/home路径时回调的函数 先默认函数没有实体
   r.GET("/home", foo)
   ```

   - 进入对应的`GET`方法

     ```go
     // 此时的relativePath="/home" handlers=foo
     func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
     	return group.handle(http.MethodGet, relativePath, handlers)
     }
     ```

   - 进入`handle`方法

     ```go
     func (group *RouterGroup) handle(httpMethod, relativePath string, handlers HandlersChain) IRoutes {
         // absolutePath得到绝对路径
     	absolutePath := group.calculateAbsolutePath(relativePath)
         // 添加对应的handlers
     	handlers = group.combineHandlers(handlers)
     	group.engine.addRoute(httpMethod, absolutePath, handlers)
     	return group.returnObj()
     }
     ```

     - 进入`group.calculateAbsolutePath(relativePath)`方法

     ```go
     func (group *RouterGroup) calculateAbsolutePath(relativePath string) string {
      	// 当前的RouterGroup是r := gin.Default()生成的 首先默认配置的group.basePath为"/"
         // relativePath = "/home"
     	return joinPaths(group.basePath, relativePath)
     }
     
     func joinPaths(absolutePath, relativePath string) string {
     	if relativePath == "" {
     		return absolutePath
     	}
     	// absolutePath = group.basePath = "/"
         // relativePath = "/home"
     	finalPath := path.Join(absolutePath, relativePath)
     	if lastChar(relativePath) == '/' && lastChar(finalPath) != '/' {
     		return finalPath + "/"
     	}
     	return finalPath
     }
     
     func Join(elem ...string) string {
     	size := 0
     	for _, e := range elem {
     		size += len(e)
     	}
     	if size == 0 {
     		return ""
     	}
         
     	buf := make([]byte, 0, size+len(elem)-1)
         
         // 此时的elem = absolutePath, relativePath
     	for _, e := range elem {
     		if len(buf) > 0 || e != "" {
     			if len(buf) > 0 {
                     // 在这里 如果我们r.GET()中设置了重复的"/"其实也无所谓 
                     // 标准的应该是不用加/ 因为这里是默认帮我们加上了
                     // 例如r.GET("/home/zyb")
                     // 这时循环结束后生成的buf此时为"/home//zyb"
                     // 但是在return Clean(string(buf))中会帮我们清除多余的"/"
     				buf = append(buf, '/')
     			}
     			buf = append(buf, e...)
     		}
     	}
     	return Clean(string(buf))
     }
     
     /*
     Code:
     paths := []string{     "a/c",     
     "a//c",     
     "a/c/.",     
     "a/c/b/..",     
     "/../a/c",     
     "/../a/b/../././/c",    
     "", }  
     for _, p := range paths {    
     	fmt.Printf("Clean(%q) = %q\n", p, path.Clean(p)) 
     } 
     Output:
     Clean("a/c") = "a/c"
     Clean("a//c") = "a/c"
     Clean("a/c/.") = "a/c"
     Clean("a/c/b/..") = "a/c"
     Clean("/../a/c") = "/a/c"
     Clean("/../a/b/../././/c") = "/a/c"
     Clean("") = "."
     */
     ```

   - 得到`aboslutePath`后 进入`group.combineHandlers(handlers)`中添加`handlers`方法

     ```go
     func (group *RouterGroup) combineHandlers(handlers HandlersChain) HandlersChain {
     	finalSize := len(group.Handlers) + len(handlers)
         // 表示当前node最大的handlers数不能超过abortIndex, abortIndex = 63
     	if finalSize >= int(abortIndex) {
     		panic("too many handlers")
     	}
     	mergedHandlers := make(HandlersChain, finalSize)
         // 相当于添加该handlers
     	copy(mergedHandlers, group.Handlers)
     	copy(mergedHandlers[len(group.Handlers):], handlers)
         // 返回新的HandlersChain
     	return mergedHandlers
     }
     ```

     

   - **重点来了 我们现在进入`group.engine.addRoute(httpMethod, absolutePath, handlers)`中**

     ```go
     // path = absoultPath = "/home"
     func (engine *Engine) addRoute(method, path string, handlers HandlersChain) {
     	// 前面说了4个method每个都只会有1个根节点 这里如果没get到则返回nil
         // 4个根节点存储在了engine.tress字段中 
         // 由于程序中r.GET("/ss")已经先生成了GET方法的root结点 因此在这里root不为nil 
         // 后面会解释为什么要先r.GET("/ss")
     	root := engine.trees.get(method)
     	if root == nil {
     		root = new(node)
     		root.fullPath = "/"
     		engine.trees = append(engine.trees, methodTree{method: method, root: root})
     	}
     	root.addRoute(path, handlers)
     }
     
     func (n *node) addRoute(path string, handlers HandlersChain) {
     	fullPath := path
     	n.priority++
     
         // 还记得我们的r.GET("/ss")吗 
         // 这里当首次为GET方法创建路由时 由于当前node为root 并存储了path为"ss"的结点 
         // 因此这里就不会直接return 方便后面的研究
     	// Empty tree
     	if len(n.path) == 0 && len(n.children) == 0 {
     		n.insertChild(path, fullPath, handlers)
     		n.nType = root
     		return
     	}
     
         // 当前node与path
     	parentFullPathIndex := 0
     
     walk:
     	for {
     		// Find the longest common prefix.
     		// This also implies that the common prefix contains no ':' or '*'
     		// since the existing key can't contain those chars.
             // 这里是寻找当前node结点 与 path的最长公共前缀
             // 注意这里的n可能不只是root结点 也可能是其他节点 后面会说明
     		i := longestCommonPrefix(path, n.path)
     
             
              // 例如 当前node的path为"home"
             // 然后我们再r.GET("/hame")添加的path为"hame"
             // 那么此时需要进行结点分裂
             // | /
             // | h
             // |_ ome
             // |_ ame
             
     		// Split edge
             // i < n.path 说明n.path与path有公共前缀 需要进行结点分裂
             // 这里涉及基数树的原理 可以在前面重新回顾
     		if i < len(n.path) {
     			child := node{
     				path:      n.path[i:],
     				wildChild: n.wildChild,
     				indices:   n.indices,
     				children:  n.children,
     				handlers:  n.handlers,
                     // 因为当前结点n的子节点即为本身 所以自减1是减去本身
     				priority:  n.priority - 1,
     				fullPath:  n.fullPath,
     			}
     			
                  // 将child添加进n的子结点中
     			n.children = []*node{&child}
     			// []byte for proper unicode char conversion, see #65
                  // 为n的indices添加n.path[i] 这里是 o
     			n.indices = bytesconv.BytesToString([]byte{n.path[i]})
                 // n.path此时重新设置为 h
     			n.path = path[:i]
                 // 不是完整的路径 "/h" 因此没有handlers 
     			n.handlers = nil
                 // 不是带参数的结点 设置false
     			n.wildChild = false
                 // fullpath 当前为 "/h"
     			n.fullPath = fullPath[:parentFullPathIndex+i]
     		}
     
     		// Make new node a child of this node
     		if i < len(path) {
                 // 这里path从ame开始 因为/hame中的/h为公共前缀
     			path = path[i:]
                 // c 为待添加结点公共前缀后的第一个字符
     			c := path[0]
     			
                 ...
                 
     		    // Check if a child with the next path byte exists
                 // 查看当前n的indices是否有与i相等 若有则回到walk继续分裂
                 // 可以配后面的图片进行理解
                 // 例如/home下有子节点abc
                 // /home
                 // ->abc
                 // 此时添加/hzmd
                 // 分裂后/h下有ome和amd和zmd三个结点
                 // /h
                 // ->ome
                 //    -->abc
                 // ->zmd
                 // 在/home路径添加omg结点 route.GET("omg")
                 // 此时继续分裂
                 // /h
                 // ->om
                 //    -->g
                 //    -->abc
                 // ->zmd
     			for i, max := 0, len(n.indices); i < max; i++ {
     				if c == n.indices[i] {
     					parentFullPathIndex += len(n.path)
     					i = n.incrementChildPrio(i)
                         // 这一步修改n为n的子结点 以便后续分裂结点
                         // 后续还是找到公共的最长前缀 而不是只有indices[i]
     					n = n.children[i]
     					continue walk
     				}
     			}
     
     			// Otherwise insert it
                 // 如果没有与当前结点的子节点有公共前缀后就直接插入
     			if c != ':' && c != '*' && n.nType != catchAll {
     				// []byte for proper unicode char conversion, see #65
     				n.indices += bytesconv.BytesToString([]byte{c})
     				child := &node{
     					fullPath: fullPath,
     				}
     				n.addChild(child)
     				n.incrementChildPrio(len(n.indices) - 1)
     				n = child
     			} else if n.wildChild {
     				// inserting a wildcard node, need to check if it conflicts with the existing wildcard
     				n = n.children[len(n.children)-1]
     				n.priority++
     
     				// Check if the wildcard matches
     				if len(path) >= len(n.path) && n.path == path[:len(n.path)] &&
     					// Adding a child to a catchAll is not possible
     					n.nType != catchAll &&
     					// Check for longer wildcard, e.g. :name and :names
     					(len(n.path) >= len(path) || path[len(n.path)] == '/') {
     					continue walk
     				}
     
     				// Wildcard conflict
     				pathSeg := path
     				if n.nType != catchAll {
     					pathSeg = strings.SplitN(pathSeg, "/", 2)[0]
     				}
     				prefix := fullPath[:strings.Index(fullPath, pathSeg)] + n.path
     				panic("'" + pathSeg +
     					"' in new path '" + fullPath +
     					"' conflicts with existing wildcard '" + n.path +
     					"' in existing prefix '" + prefix +
     					"'")
     			}
     
     			n.insertChild(path, fullPath, handlers)
     			return
     		}
     
     		// Otherwise add handle to current node
     		if n.handlers != nil {
     			panic("handlers are already registered for path '" + fullPath + "'")
     		}
     		n.handlers = handlers
     		n.fullPath = fullPath
     		return
     	}
     }
     ```

     具体图

     ![](D:\Program Files\电子书\go\md\图片\image-20220904010006821.png)

     ![](D:\Program Files\电子书\go\md\图片\image-20220904010406356.png)

3. 至此 我们就可以`return group.returnObj()` 即`RouterGroup`本身啦 最后在定义的`engine`结构体中的`trees`字段我们可以查到各个方法的`root's node` 再从`root`的`children`字段中我们就可以找到定义了的路由及其对应的回调函数了