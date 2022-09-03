# Shell编程

1. Shell环境

   - `#! `是一个约定好的标记 用来告诉Linux系统这个shell脚本需要用什么解释器来执行
   - **解释器**：又译为直译器，是一种电脑程序，能够把高级[编程语言](https://baike.baidu.com/item/编程语言?fromModule=lemma_inlink)一行一行直接转译运行。解释器不会一次把整个程序转译出来，只像一位“中间人”，每次运行程序时都要先转成另一种语言再作运行，因此解释器的程序运行速度比较缓慢。它每转译一行程序叙述就立刻运行，然后再转译下一行，再运行，如此不停地进行下去

   ```shell
   #!/bin/bash
   echo "hello world"
   ```

   - 运行shell脚本的两种方法

     - 作为可执行程序

       ```shell
       # 先赋予可执行权限
       chmod +x ./name.sh
       # 执行脚本程序
       ./name.sh 
       ```

       

     - 作为解释器参数 **这种方式不需要在脚本程序的第一行中指定解释器信息**

       ```shell
       /bin/sh name.sh
       /bin/bash name.sh
       ```

# 变量

1. 定义变量

   - `=`号之间不允许有空格
   - 定义变量不需要美元符号
   - 使用变量才需要美元符号

   ```shell
   var_name="zyb"
   
   echo $var_name
   ```

   

2. **只读变量**

   - 使用`readonly`关键字可以将变量指定为只读变量 只读变量的值不允许修改

   ```shell
   #!/bin/bash
   my="zyb"
   readonly my
   
   ```

   ```shell
   # 修改my变量
   my="zyb1"
   
   # 输出
   # 解释器执行到readonly my时出错
   /bin/bash: my: This variable is read only.
   ```

   

3. **删除变量**

   - `unset`命令不能删除只读变量
   - 变量被删除后不能被重新赋值使用 也不能输出原有值

   ```shell
   unset var_name
   ```

   

   ```shell
   #!/bin/bash
   my="zyb"
   unset my
   echo $my
   
   # 上述将没有输出
   
   ```

   

4. **字符串**

   - 字符串使用双引号
     - **双引号中可以指定使用变量**
     - 双引号中可以出现转义字符

   ```shell
   #!/bin/bash
   str="zyb"
   # 将双引号转义
   # 用美元符号设置变量
   this="hello \" $str \" \n"
   
   # 输出
   hello " zyb "
   ```

   - 获取字符串长度

     ```shell
     string="abcd"
     echo ${#string} # 输出 4
     ```

   - 提取子字符串

     ```shell
     strint="abcd"
     echo${string:1:4} # 从第第二个字符开始截取4个字符 即2-5
     ```

     

# Shell数组

1. bash仅支持一维数组
2. 下标也是从0开始
3. 元素之间不需要加逗号

```shell
arr_name=(value1  value2  value3...)	
```

- 读取数组

  ```shell
  ${数组名[下标]}
  
  # 读取第一个元素
  value=${arr_name[0]}
  ```

- 使用`@`符号获取数组中的所有元素

  ```shell
  echo ${arr_name[@]}
  ```

  

- 获取数组的长度

  ```shell
  echo ${#arr_name[@]}
  echo ${#arr_name[*]}
  
  # 取数组中下标为n的元素的数据长度
  echo ${#arr_name[n]}
  ```

4. **关联数组**

   - 使用`declare -A `来定义关联数组 即可以使用任意字符串、或者整数作为下标来访问数组元素

   ```shell
   #!/bin/bash
   declare -A arr=(
   ["zyb"]="zxzx"
   ["rub"]="zybs"
   ["xza"]="sae"
   )
   echo ${arr["zyb"]}
   
   # 输出
   # zxzx
   ```

   - 获取数组的所有键 添加`!`号

   ```shell
   echo ${!arr_name[@]}
   echo ${!arr_name[*]}
   ```

   

# 注释

1. 用`#`开头的行即注释 会被解释器忽略

2. **多行注释**

   ```shell
   :<<EOF
   注释内容...
   注释内容...
   注释内容...
   EOF
   
   :<<'
   注释内容...
   注释内容...
   注释内容...
   '
   
   :<<!
   注释内容...
   注释内容...
   注释内容...
   !
   ```



# 传递参数

1. 在终端执行脚本时可以附带参数 第一个参数0默认为执行脚本的命令 

   ```shell
   echo "Shell 传递参数实例！";
   echo "第一个参数为：$1";
   echo "第二个参数为：$2";
   echo "第三个参数为：$3";
   
   ./test.sh zyb 22 33
   # 输出
   # 第一个参数为：zyb
   # 第二个参数为：22
   # 第三个参数为：33
   ```

   

   | 参数处理 | 说明                                                         |
   | -------- | ------------------------------------------------------------ |
   | $#       | 传递到脚本的参数个数                                         |
   | $*       | 以一个单字符串显示所有向脚本传递的参数。                     |
   | $@       | 与$*相同，但是使用时加引号，并在引号中返回每个参数。         |
   | $$       | 脚本运行的当前进程ID号                                       |
   | $!       | 后台运行的最后一个进程的ID号                                 |
   | $?       | 显示最后命令的退出状态。0表示没有错误，其他任何值表明有错误。 |
   | $-       | 显示Shell使用的当前选项，与set命令功能相同。                 |

   - 测试

     ```shell
     echo "传递参数"
     echo " # : $#"
     echo " * : $*"
     echo " @ : $@"
     echo " $ : $$"
     echo " ! : $!"
     echo " ? : $?"
     echo " - : $-"
     ```

     ![image-20220831150448081](D:\Program Files\电子书\go\md\图片\image-20220831150448081.png)

