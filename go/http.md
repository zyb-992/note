# net/http包

1. 首先一个最简单的Web例程 

   ```go
   package main
   
   import (
     "fmt"
     "net/http"
   )
   
   func index(w http.ResponseWriter, r *http.Request) {
     fmt.Fprintln(w, "Hello World")
   }
   
   func main() {
     http.HandleFunc("/", index)
     http.ListenAndServe(":8080", nil)
   }
   ```

   

   - 程序通过HandleFunc函数注册了**路径**对应的**处理函数**
   - 这里是注册到Go官方默认的DefaultServeMux中

   ```go
   func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
   	DefaultServeMux.HandleFunc(pattern, handler)
   }
   
   // DefaultServeMux is the default ServeMux used by Serve.
   var DefaultServeMux = &defaultServeMux
   var defaultServeMux ServeMux
   type ServeMux struct {
   	mu    sync.RWMutex
   	m     map[string]muxEntry
   	es    []muxEntry // slice of entries sorted from longest to shortest.
   	hosts bool       // whether any patterns contain hostnames
   }
   type muxEntry struct {
       h       Handler
       pattern string
   }
   
   // DefaultServeMux.HandleFunc(pattern, handler)
   // HandleFunc registers the handler function for the given pattern.
   func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
   	if handler == nil {
   		panic("http: nil handler")
   	}
   	mux.Handle(pattern, HandlerFunc(handler))
   }
   
   // Handle registers the handler for the given pattern.
   // If a handler already exists for pattern, Handle panics.
   func (mux *ServeMux) Handle(pattern string, handler Handler) {
       // if e not exists in m
       // m 是 map 键：路径 值：对应的muxEntry(处理函数HandlerFunc以及路径pattern)
   	e := muxEntry{h: handler, pattern: pattern}
   	mux.m[pattern] = e
   }
   ```

![image-20220810131220595](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220810131220595.png)

- **将路径及其对应处理函数注册到DefaultServeMux中以后** 接着调用http.ListenAndServe()

  ```go 
  http.ListenAndServe(":8080", nil)
  
  // ListenAndServe listens on the TCP network address addr and then calls
  // Serve with handler to handle requests on incoming connections.
  // Accepted connections are configured to enable TCP keep-alives.
  // The handler is typically nil, in which case the DefaultServeMux is used.
  // ListenAndServe always returns a non-nil error.
  func ListenAndServe(addr string, handler Handler) error {
  	server := &Server{Addr: addr, Handler: handler}
  	return server.ListenAndServe()
  }
  
  // A Server defines parameters for running an HTTP server.
  // The zero value for Server is a valid configuration.
  type Server struct {
  	···
  	Addr string		// 服务器Server的IP地址
  	Handler Handler // handler to invoke, http.DefaultServeMux if nil
  }
  
  // server.ListenAndServe()
  // ListenAndServe listens on the TCP network address srv.Addr and then
  // calls Serve to handle requests on incoming connections.
  // Accepted connections are configured to enable TCP keep-alives.
  // If srv.Addr is blank, ":http" is used.
  // ListenAndServe always returns a non-nil error. After Shutdown or Close,
  // the returned error is ErrServerClosed.
  func (srv *Server) ListenAndServe() error {
  	if srv.shuttingDown() {
  		return ErrServerClosed
  	}
  	addr := srv.Addr
  	if addr == "" {
  		addr = ":http"
  	}
  	ln, err := net.Listen("tcp", addr)
  	if err != nil {
  		return err
  	}
  	return srv.Serve(ln)
  }
  
  // srv.Serve(ln)
  func (srv *Server) Serve(l net.Listener) error {
  	···对于错误或者时间判断
      baseCtx := context.Background()
  	···
  	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
  	for {
          // 当客户端连接到Server 接收返回rw(Conn)
  		rw, err := l.Accept()
  		// ···错误判断
          // 上下文传递 每个客户端连接都有对应上下文
  		connCtx := ctx
  		if cc := srv.ConnContext; cc != nil {
  			connCtx = cc(connCtx, rw)
  			if connCtx == nil {
  				panic("ConnContext returned nil")
  			}
  		}
  		tempDelay = 0
  		c := srv.newConn(rw)
  		c.setState(c.rwc, StateNew, runHooks) // before Serve can return
          // 开启协程进行服务
  		go c.serve(connCtx)
  	}
  }
  
  // c.serve(connCtx)
  func (c *conn) serve(ctx context.Context) {
    for {
      w, err := c.readRequest(ctx)
        // w：response结构体 实现了ResponseWriter接口
      serverHandler{c.server}.ServeHTTP(w, w.req)
      w.finishRequest()
    }
  }
  
  // serverHandler{c.server}.ServeHTTP(w, w.req)
  type serverHandler struct {
      srv *Server
  }
  func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
      // handler是Handler接口类型
  	handler := sh.srv.Handler
      // 这里例程并没有注册Handler(nil)
      // http.ListenAndServe(":8080", nil)
  	if handler == nil {
          // 使用Go指定的DefaultServeMux多路复用器
  		handler = DefaultServeMux
  	}
  	if req.RequestURI == "*" && req.Method == "OPTIONS" {
  		handler = globalOptionsHandler{}
  	}
  	···
      // DefaultServeMux.ServeHTTP(rw, req)
  	handler.ServeHTTP(rw, req)
  }
  
  // ServeHTTP dispatches the request to the handler whose
  // pattern most closely matches the request URL.
  func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
  	if r.RequestURI == "*" {
  		if r.ProtoAtLeast(1, 1) {
  			w.Header().Set("Connection", "close")
  		}
  		w.WriteHeader(StatusBadRequest)
  		return
  	}
  	h, _ := mux.Handler(r)
  	h.ServeHTTP(w, r)
  }
  
  // h, _ := mux.Handler(r)
  // 根据客户端的消息Request中的字段来得到host以及path
  // 然后调用mux.handler(host, r.URL.Path)查找客户端请求的Path对应的路由处理函数
  func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
  
  	// CONNECT requests are not canonicalized.
  	if r.Method == "CONNECT" {
  		// If r.URL.Path is /tree and its handler is not registered,
  		// the /tree -> /tree/ redirect applies to CONNECT requests
  		// but the path canonicalization does not.
  		if u, ok := mux.redirectToPathSlash(r.URL.Host, r.URL.Path, r.URL); ok {
  			return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
  		}
  
  		return mux.handler(r.Host, r.URL.Path)
  	}
  
  	// All other requests have any port stripped and path cleaned
  	// before passing to mux.handler.
  	host := stripHostPort(r.Host)
  	path := cleanPath(r.URL.Path)
  
  	// If the given path is /tree and its handler is not registered,
  	// redirect for /tree/.
  	if u, ok := mux.redirectToPathSlash(host, path, r.URL); ok {
  		return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
  	}
  
  	if path != r.URL.Path {
  		_, pattern = mux.handler(host, path)
  		u := &url.URL{Path: path, RawQuery: r.URL.RawQuery}
  		return RedirectHandler(u.String(), StatusMovedPermanently), pattern
  	}
  
  	return mux.handler(host, r.URL.Path)
  }
  
  // handler is the main implementation of Handler.
  // The path is known to be in canonical form, except for CONNECT methods.
  func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
  	mux.mu.RLock()
  	defer mux.mu.RUnlock()
  
  	// Host-specific pattern takes precedence over generic ones
      // 是否注册了根目录 “/”
  	if mux.hosts {
  		h, pattern = mux.match(host + path)
  	}
  	if h == nil {
  		h, pattern = mux.match(path)
  	}
  	if h == nil {
  		h, pattern = NotFoundHandler(), ""
  	}
  	return
  }
  ```

  

