# 标准库

> IO https://mp.weixin.qq.com/s/nyD8B6BGZ_Yy7UqtTQ3ybw

## strings标准库

1. `HasPrefix`

   - 判断s是否有前缀字符串prefix

   > ```go
   > func HasPrefix(s, prefix string) bool
   > ```

   ```go
   ```

   

2. `HasSuffix`

   - 判断s是否有后缀字符串suffix

   > ```go
   > func HasSuffix(s, suffix string) bool
   > ```

   ```go
   
   ```

   

3. `Contains`

   - 判断字符串s是否包含子字符串substr

   > ```go
   > func Contains(s, substr string) bool
   > ```

   ```go
   ```

   

4. `ContainsAny`

   - 判断字符串s是否包含字符串chars中的任一字符	

   > ```go
   > func ContainsAny(s, chars string) bool
   > ```

   ```go
   ```

   

5. `Count`

   - 返回字符串中有几个不重复的sep字串

   > ```go
   > func ContainsAny(s, chars string) bool
   > ```

   ```go
   ```




6. `ReplaceAll`

   - 将s中出现的old字符串替换为new字符串

   > ```go
   > func ReplaceAll(s, old, new string) string 
   > ```

## bytes标准库

```go
// A Reader implements the io.Reader, io.ReaderAt, io.WriterTo, io.Seeker,
// io.ByteScanner, and io.RuneScanner interfaces by reading from
// a byte slice.
// Unlike a Buffer, a Reader is read-only and supports seeking.
// The zero value for Reader operates like a Reader of an empty slice.
type Reader struct {
	s        []byte
	i        int64 // current reading index
	prevRune int   // index of previous rune; or < 0
}


// A Buffer is a variable-sized buffer of bytes with Read and Write methods.
// The zero value for Buffer is an empty buffer ready to use.
type Buffer struct {
	buf      []byte // contents are the bytes buf[off : len(buf)]
	off      int    // read at &buf[off], write at &buf[len(buf)]
	lastRead readOp // last read operation, so that Unread* can work correctly.
}
```





## Strconv标准库

1. `Atoi` 将输入的字符串转为整数并返回

   > **func Atoi(s string) (i int, err error)**

   - **输入的字符串只允许是数字型的字符串 不允许有空格或其他字符**

   - 若有其他字符则返回空字符串

     ```go
     s := "21311"
     i, _ := strconv.Atoi(s)
     // Switch string to int
     fmt.Printf("type : %T, value : %v\n", i, i)
     
     // 输出
     type : int, value : 21311
     ```
     
     

2. `Itoa`将输入的整数转为字符串并返回

   > **func Itoa(i int) string**

   ```go
   i := 10
   s := strconv.Itoa(i)
   fmt.Println(s)
   
   // 输出
   "10"
   ```

   



# 与IO相关的标准包

> **`bufio.Reader`是结构体，而`io.Reader`是接口类型**

## io包

1. io中相关类型

   ```go
   // Read方法会接收一个字节数组p，并将读取到的数据存进该数组，最后返回读取的字节数n。
   // 注意n不一定等于读取的数据长度，比如字节数组p的容量太小，n会等于数组的长度
   type Reader interface {
       Read(p []byte) (n int, err error)
   }
   
   // Write 方法同样接收一个字节数组p，并将接收的数据保存至文件或者标准输出等，返回的n表示写入的数据长度。
   // 当n不等于len(p)时，返回一个错误。
   type Writer interface {
       Write(p []byte) (n int, err error)
   }
   
   // 关闭操作
   type Closer interface {
       Close() error
   }
   ```

   

## ioutil包

1. `ReadAll`

   - 一次性读取Reader中的所有内容，并返回存储该内容的字节切片
   - 内部使用了bytes.Buffer从io.Reader中读取所有内容

   > **func ReadAll(r io.Reader) ([]byte, error)**

   ```go
   // Go语言中File实现了Reader和Writer接口
   // 打开文件，返回文件实例File指针
   file, err := os.Open("1.txt")  
   if err != nil {
     fmt.Println(err)
   }
   // 从Reader中读取文件中所有内容到data中
   data, err := ioutil.ReadAll(file)
   if err != nil {
     fmt.Println(err)
   } else {
     // 将字节切片转换为字符串输出
     fmt.Println(string(data))
   }
   ```

   

2. `ReadFile`

   - 一次性从参数文件路径表示的文件中读取所有的文件内容 返回存储读取内容的字节切片
   - 内部使用了bytes.Buffer从os.Open()返回的Reader中读取所有内容

   > **func ReadFile(filename string) ([]byte, error)**

   ```go
   data, err = ioutil.ReadFile("./ioutil/1.txt")
   	if err != nil {
   		fmt.Println(err)
   	} else {
   		fmt.Println(string(data))
   	}
   ```

   

3. `ReadDir`

   - 从输入参数文件夹名中读取其目录下的所有文件和目录 返回FileInfo切片
   - `os.File`结构体实现了`FileInfo`接口
   - `ReadDir`内部使用了os包中的`File.Readdir`来读取目录下的所有文件和子目录，还会对结果进行排序

   > **func ReadDir(dirname string) ([]fs.FileInfo, error)**

   ```go
   // func ReadDir(dirname string) ([]fs.FileInfo, error)
   	fileInfos, err := ioutil.ReadDir("./ioutil/in")
   	if err != nil {
   		fmt.Println(err)
   	}
   	for index, fileInfo := range fileInfos {
   		// the hide files are included
   		fmt.Println(index, fileInfo.Name())
   	}
   
   ```

   

   ## bufio包

   ```go
   // Reader implements buffering for an io.Reader object.
   type Reader struct {
   	buf          []byte
   	rd           io.Reader // reader provided by the client
   	r, w         int       // buf read and write positions
   	err          error
   	lastByte     int // last byte read for UnreadByte; -1 means invalid
   	lastRuneSize int // size of last rune read for UnreadRune; -1 means invalid
   }
   ```

   

   1. `NewReader`

      - 使用`bufio.NewReader(rd io.Reader) *Reader`创建一个带缓冲区（缓冲区默认大小**4096**字节）的`bufio.Reader`实例，然后使用该Reader读取文件内容

      > **bufio.NewReader(rd io.Reader) \*Reader**

      ```go
      f, err := os.Open("./bufio/connect.txt")
      defer f.Close()
      
      if err != nil {
          panic(err)
      }
      //func NewReader(rd io.Reader) *Reader
      // retrurn bufio.Reader
      reader := bufio.NewReader(f)
      totLine := 0
      for {
          //func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)
          content, isPrefix, err := reader.ReadLine()
          fmt.Println(string(content))
          if !isPrefix {
              totLine++
          }
      
          if err == io.EOF {
              fmt.Println("All ", totLine, " row contents")
              break
          }
      }
      
      // 输出
      package main
      
      import "fmt"
      
      func main() {
              fmt.Println("hello world")
      }
      
      All  8  row contents
      
      ```

   2. `ReadLine`

      > **func (b *Reader) ReadLine() (line []byte, isPrefix bool, err error)**

      - ReadLine是一个低水平的行数据读取原语。大多数调用者应使用ReadBytes('\n')或ReadString('\n')代替，或者使用Scanner。

      - ReadLine尝试返回一行数据，不包括行尾标志的字节。**如果行太长超过了缓冲（默认4096)，返回值isPrefix会被设为true，并返回行的前面一部分。该行剩下的部分将在之后的调用中返回。返回值isPrefix会在返回该行最后一个片段时才设为false。**返回切片是缓冲的子切片，只在下一次读取操作之前有效。ReadLine要么返回一个非nil的line，要么返回一个非nil的err，两个返回值至少一个非nil。

      - 返回的文本不包含行尾的标志字节（"\r\n"或"\n"）。如果输入流结束时没有行尾标志字节，方法不会出错，也不会指出这一情况。在调用ReadLine之后调用UnreadByte会总是吐出最后一个读取的字节（很可能是该行的行尾标志字节），即使该字节不是ReadLine返回值的一部分。

        

   3. `ReadBytes`

      > func (b *Reader) ReadBytes(delim byte) (line []bye, err error)

      - **ReadBytes读取直到第一次遇到delim字节，返回一个包含已读取的数据和delim字节的切片。**如果ReadBytes方法在读取到delim之前遇到了错误，它会返回在错误之前读取的数据以及该错误（一般是io.EOF）。当且仅当ReadBytes方法返回的切片不以delim结尾时，会返回一个非nil的错误。

        

   4. `ReadString`

      - `ReadString`方法从`bufio.Reader`中读取字符串，直到读到参数`delim`，就返回`delim`和之前的字符串。如果将`delim`设置为`\n`，相当于按行来读取了

      > **ReadString(delim byte) (string, error)**

   

   4. **`bufio.Reader`是结构体，而`io.Reader`是接口类型**

      

   5. `NewWriter`

      ```go
      path := "./bufio/test.txt"
      // 创建文件
      f, err = os.Create(path)
      // 写入部分数据
      f.Write([]byte{'z', 'y', 'b'})
      defer f.Close()
      if err != nil {
          panic(err)
      }
      
      // 新建bufio.Writer结构体
      bufferWrite := bufio.NewWriter(f)
      if err != nil {
          panic(err)
      }
      // 待写入的数据
      demo := "1244340934"
      for _, v := range demo {
          // write the data(demo) to buffer(bufferWrite.buf)
          bufferWrite.WriteString(string(v))
      }
      // 读取文件内容 此时为zyb
      data, _ := ioutil.ReadFile(path)
      fmt.Println(string(data))
      
      // 将bufio.Writer中缓冲区的数据写入文件中
      //Flush writes any buffered data to the underlying io.Writer.
      bufferWrite.Flush()
      
      // 再次读取 此时为zyb1244340934
      data, _ = ioutil.ReadFile(path)
      fmt.Println(string(data))
      ```

      

   6. `Buffered`

      > func (*Reader) [Buffered](https://github.com/golang/go/blob/master/src/bufio/bufio.go?name=release#261)

      - Buffered返回缓冲中现有的可读取的字节数

   7. `ReadByte`

      > func (b *Reader) ReadByte() (c byte, err error)

      - 读取并返回一个字节 没有可返回的数据则返回错误

   8. 