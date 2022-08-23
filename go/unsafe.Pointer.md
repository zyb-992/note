# Unsafe.Pointer

> [详解 Go 团队不建议用的 unsafe.Pointer - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/425418907)

1. Go中不同类型的指针不能相互转换
   
   ```go
   // 错误示例
   func main(){
       num := 4
       numPointer := &num
   
       flnum := (*float32)(numPointer)
       fmt.Println(flunm)
   }    
   
    // 报错
   connot convert numPointer(type *int) to type *float32
   ```
   
2. **unsafe.Pointer**：
   
   1. 表示任意类型且可寻址的指针值
   
   2. 可以在不同类型之间相互进行转换
   
   3. 核心操作
      
      > - 任何类型的指针值都可以转换为 Pointer。  
      > 
      > - Pointer 可以转换为任何类型的指针值。  
      > 
      > - uintptr 可以转换为 Pointer。  
      > 
      > - Pointer 可以转换为 uintptr。
   
   4. 示例
      
      1. ```go
         func main(){
          num := 5
          numPointer := &num
         
          flnum := (*float32)(unsafe.Pointer(numPointer))
          fmt.Println(flnum)
         }
         ```
   
   5. 在转换时 因为不知道此时的unsafe.Pointer指向的具体类型 因此不能对它进行操作

3. **unsfae.Offsetof**
   
   1. 针对于结构体使用
   
   2. unsafe.Offsetof：返回成员变量 x 在结构体当中的偏移量。更具体的讲，就是返回结构体初始位置到 x 之间的字节数。需要注意的是入参 `ArbitraryType` 表示任意类型，并非定义的 `int`。它实际作用是一个占位符
   
   3. `uintptr` 类型是不能存储在临时变量中的。因为从 GC 的角度来看，`uintptr` 类型的临时变量只是一个无符号整数，并不知道它是一个指针地址

4. 普通指针具体类型就是`uintptr`

