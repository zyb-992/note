# Go第三方包

# Viper

> [Go语言配置管理神器——Viper中文教程 | 李文周的博客 (liwenzhou.com)](https://www.liwenzhou.com/posts/Go/viper_tutorial/#autoid-0-0-0)

1. [Viper](https://github.com/spf13/viper)是适用于Go应用程序的完整配置解决方案。它被设计用于在应用程序中工作，并且可以处理所有类型的配置需求和格式。

   - **Viper是适用于Go应用程序（包括`Twelve-Factor App`）的完整配置解决方案。它被设计用于在应用程序中工作，并且可以处理所有类型的配置需求和格式。**它支持以下特性：
     - 设置默认值
     - 从`JSON`、`TOML`、`YAML`、`HCL`、`envfile`和`Java properties`格式的配置文件读取配置信息
     - 实时监控和重新读取配置文件（可选）
     - 从环境变量中读取
     - 从远程配置系统（etcd或Consul）读取并监控配置变化
     - 从命令行参数读取配置
     - 从buffer读取配置
     - 显式配置值

2. 安装

   ```go
   go get github.com/spf13/viper
   ```

3. Viper可以读取从多个地方读取配置文件 **同时可以使用结构体变量保存配置信息**

   ```go
   viper.SetConfigFile("./config.yaml") // 指定配置文件路径
   viper.SetConfigName("config") // 配置文件名称(无扩展名)
   viper.SetConfigType("yaml") // 如果配置文件的名称中没有扩展名，则需要配置此项
   viper.AddConfigPath("/etc/appname/")   // 查找配置文件所在的路径
   viper.AddConfigPath("$HOME/.appname")  // 多次调用以添加多个搜索路径
   viper.AddConfigPath(".")               // 还可以在工作目录中查找配置
   
   // 读取配置文件
   err := viper.ReadInConfig() // 查找并读取配置文件
   if err != nil { // 处理读取配置文件的错误
   	panic(fmt.Errorf("Fatal error config file: %s \n", err))
   }
   ```

4. 监控并重新读取配置文件

   1. 当运行过程中修改了配置文件 可以在不重启服务的情况下重新读取配置文件

      ```go
      // 在WatchConfig中开启Goroutine运行 并发判断配置文件中是否有被修改
      viper.WatchConfig()
      
      viper.OnConfigChange(func(e fsnotify.Event) {
        // 配置文件发生变更之后会调用的回调函数
      	fmt.Println("Config file changed:", e.Name)
      })
      ```

      

5. 将配置文件保存到结构体各字段信息中

   ```go
   // viper 已经ReadInConfig
   // 根据结构体标签来写入字段信息
   if err := viper.Unmarshal(_Struct_); err != nil {
   		panic(fmt.Errorf("unmarshal conf failed, err:%s \n", err))
   }
   ```

   

# Zap

> [在Go语言项目中使用Zap日志库 | 李文周的博客 (liwenzhou.com)](https://www.liwenzhou.com/posts/Go/zap/)

1. [Zap](https://github.com/uber-go/zap)是非常快的、结构化的，分日志级别的Go日志库。

2. 安装

   ```go
   go get -u go.uber.org/zap
   ```

   

3.  两种日志记录器

   - **SugarLogger**
   - **Logger**

   ```go
   // 通过调用zap.NewProduction()/zap.NewDevelopment()或者zap.Example()创建一个Logger 不同函数记录的信息不同
   var logger *zap.Logger
   
   func main() {
   	InitLogger()
     defer logger.Sync()
   	simpleHttpGet("www.google.com")
   	simpleHttpGet("http://www.google.com")
   }
   
   func InitLogger() {
   	logger, _ = zap.NewProduction()
   }
   
   func simpleHttpGet(url string) {
   	resp, err := http.Get(url)
   	if err != nil {
   		logger.Error(
   			"Error fetching url..",
   			zap.String("url", url),
   			zap.Error(err))
   	} else {
   		logger.Info("Success..",
   			zap.String("statusCode", resp.Status),
   			zap.String("url", url))
   		resp.Body.Close()
   	}
   }
   
   // 输出
   {"level":"error","ts":1572159218.912792,"caller":"zap_demo/temp.go:25",
    "msg":"Error fetching url..","url":"www.sogo.com",
    "error":"Get www.sogo.com: unsupported protocol scheme \"\"",
   "stacktrace":"main.simpleHttpGet\n\t/Users/q1mi/zap_demo/temp.go:25\nmain.main\n\t/Users/q1mi/zap_demo/temp.go:14\nruntime.main\n\t/usr/local/go/src/runtime/proc.go:203"}
   
   {"level":"info","ts":1572159219.1227388,"caller":"zap_demo/temp.go:30",
    "msg":"Success..","statusCode":"200 OK","url":"http://www.sogo.com"}
   ```

4. 定制Logger

   ```go
   // 将日志写入到文件中而不是打印到终端
   func New(core zapcore.Core, options ...Option) *Logger
   
   // zapcore.Core需要三个配置：Encoder, WriteSyncer, LogLevel
   // 先配置zapcore.Core 再New中放入生成的core然后返回Logger
   ```

   

5. zap的LogLevel级别

   ```go
   const (
   	// DebugLevel logs are typically voluminous, and are usually disabled in
   	// production.
   	DebugLevel = zapcore.DebugLevel
   	// InfoLevel is the default logging priority.
   	InfoLevel = zapcore.InfoLevel
   	// WarnLevel logs are more important than Info, but don't need individual
   	// human review.
   	WarnLevel = zapcore.WarnLevel
   	// ErrorLevel logs are high-priority. If an application is running smoothly,
   	// it shouldn't generate any error-level logs.
   	ErrorLevel = zapcore.ErrorLevel
   	// DPanicLevel logs are particularly important errors. In development the
   	// logger panics after writing the message.
   	DPanicLevel = zapcore.DPanicLevel
   	// PanicLevel logs a message, then panics.
   	PanicLevel = zapcore.PanicLevel
   	// FatalLevel logs a message, then calls os.Exit(1).
   	FatalLevel = zapcore.FatalLevel
   )
   ```

6. 将日志输出到多个位置

   ```go
   func getLogWriter() zapcore.WriteSyncer {
   	file, _ := os.Create("./test.log")
   	// 利用io.MultiWriter支持文件和终端两个输出目标
   	ws := io.MultiWriter(file, os.Stdout)
   	return zapcore.AddSync(ws)
   }
   ```

   

7. 将日志单独输出到文件

   ```go
   func InitLogger() {
   	encoder := getEncoder()
       
   	// test.log记录全量日志
   	logF, _ := os.Create("./test.log")
   	c1 := zapcore.NewCore(encoder, zapcore.AddSync(logF), zapcore.DebugLevel)
       
   	// test.err.log记录ERROR级别的日志
   	errF, _ := os.Create("./test.err.log")
   	c2 := zapcore.NewCore(encoder, zapcore.AddSync(errF), zap.ErrorLevel)
       
   	// 使用NewTee将c1和c2合并到core
   	core := zapcore.NewTee(c1, c2)
   	logger = zap.New(core, zap.AddCaller())
   }
   ```

   

   # Validator
   
   > Go每日一库 [Go 每日一库之 validator - 大俊的博客 (darjun.github.io)](https://darjun.github.io/2020/04/04/godailylib/validator/)
   >
   > GitHub https://github.com/go-playground/validator
   >
   > 官方文档 https://pkg.go.dev/github.com/go-playground/validator/v10#FieldError

1. 