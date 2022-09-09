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
       sh name.sh	
       ```

# 变量

1. 定义变量

   - `=`号之间不允许有空格
   - 定义变量不需要美元符号
   - 使用变量才需要美元符号
   - **区分大小写**

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

   
   5. **命令替换**
   
      - 将命令输出赋给变量
        - 反引号字符
        - $()格式
   
      ```shell
      testing=`date`
      testing=${date}
      
      i=`expr 1 + 5`
      i=$(expr 1 + 5) 
      ```



# 执行数学运算

1. `expr` 

   - 中间必须有分号

     ```shell
     `expr 1 + 5` 
     
     # no
     `expr 1+5`
     ```

   ![](D:\Program Files\电子书\go\md\图片\image-20220908200957906.png)

2. 使用方括号[]

   ```shell
   var1=100
   var2=50
   var3=45
   var4=$[$var1 * ($var2 - $var3)]
   echo the final result is $var4
   ```

   - bash shell数学运算符只支持整数运算 不支持浮点数
   - 解决bash中数学运算限制 使用内奸的bash计算器 : **bc**

# 退出脚本

1. 查看状态码

   ```shell
   echo $?
   ```

   

2. 查看退出状态码

   - **正常退出状态码为0**

   ```shell
   ls -l
   ...
   echo $? 
   0 # 正常退出状态码为0
   ```

3. 无效命令状态码为正值 状态码为127

![](D:\Program Files\电子书\go\md\图片\image-20220908202134463.png)

4. `exit`命令
   1. exit命令允许在脚本结束时指定一个退出状态码
   2. 查看脚本的退出码时 会得到作为参数传给exit命令的值
   3. **退出状态码最大只能是255**

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

# 流程控制

1. `test`

   - **test 命令用于检查某个条件是否成立，它可以进行数值、字符和文件三个方面的测试。**

     - **数值测试**

       | 参数 | 说明           |
       | :--- | :------------- |
       | -eq  | 等于则为真     |
       | -ne  | 不等于则为真   |
       | -gt  | 大于则为真     |
       | -ge  | 大于等于则为真 |
       | -lt  | 小于则为真     |
       | -le  | 小于等于则为真 |

     - **字符串测试**

       - 比较字符串时 大小写符号必须进行转义 否则会被认为重定向符号 把字符串值当作文件名

         ```shell
         var1="1"
         var2="2"
         if test [ $var1 > $var2] # 导致输出重定向问题
         if test [ $var1 \> $var2] # 正确
         ```

         - 在test中 大写字母默认是小于小写字母的 比较测试中使用的是标准的ASCII顺序


       | 参数      | 说明                     |
       | --------- | ------------------------ |
       | =         | 等于则为真               |
       | !=        | 不相等则为真             |
       | -z 字符串 | 字符串的长度为零则为真   |
       | -n 字符串 | 字符串的长度不为零则为真 |
    
     - **文件测试**
    
       | 参数                |           | 说明                                 |
       | :------------------ | --------- | :----------------------------------- |
       | -e 文件名           | exist     | 如果文件存在则为真                   |
       | -r 文件名           | read      | 如果文件存在且可读则为真             |
       | -w 文件名           | write     | 如果文件存在且可写则为真             |
       | -x 文件名           | execute   | 如果文件存在且可执行则为真           |
       | -s 文件名           | least     | 如果文件存在且至少有一个字符则为真   |
       | -d 文件名           | directory | 如果文件存在且为目录则为真           |
       | -f 文件名           | file      | 如果文件存在且为普通文件则为真       |
       | -c 文件名           | char      | 如果文件存在且为字符型特殊文件则为真 |
       | -b 文件名           | block     | 如果文件存在且为块特殊文件则为真     |
       | 文件名1 -nt 文件名2 | new than  | 判断文件1是否比文件2日期上更新       |

   - Shell 还提供了与( -a )、或( -o )、非( ! )三个逻辑操作符用于将测试条件连接起来，其优先级为： **!** 最高， **-a** 次之， **-o** 最低。

     ```shell
     cd /bin
     if test -e ./1.txt -o -e ./2.txt
     then 
     	echo "exits"
     else
     	echo "no exits"
     fi
     
     # []之间用空格隔开中括号和命令
     if  [ -e ./1.txt -o -e ./2.txt ]
     then 
     	echo "exits"
     else
     	echo "no exits"
     fi
     ```
     
   - 在if中使用test命令确定变量中是否有内容，有内容则会返回0 没有内容则出现相反情况

     ```shell
     i=""
     if test $i # exit not 0
     then 
     	echo "no null"
     fi	
     ```

     

     

2. `if` 

   - `if`根据条件中的退出状态码来判断执行哪条语句 
   - 状态码为0时即执行`then`后面的语句 不为0则执行`else`或者`elif再次判断`

   ```shell
   if condition
   then
   	...
   elif condition
   then
   	...
   else
   	...
   fi
   ```

   - `if`和`test`连用
     - if中条件判断一般都是shell命令 
     - test语句可以在if中测试其他条件


   ```shell
   if test [$a -eq $b]
   then
   	...
   else
   	...
   fi
   ```

   - 使用(())作为判断语句

   ```shell
   if (( $a == $b ))
   then 
   	echo "a = b"
   else
   	echo "a != b"
   fi
   ```

   - 使用 `&&` 或者 `||`来进行组合命令测试

     ```shell
     if [ -d $HOME ] && [ -w $HOME/testing]
     then
     	...
     else
     	...
     fi
     ```

   - 在if中使用双括号命令 能在比较过程中使用更高级的表达式

     ```shell
     if (( expression ))
     ```

     ![image-20220909111405830](D:\Program Files\电子书\go\md\图片\image-20220909111405830.png)

   - 使用双方括号命令提供针对字符串比较的高级特性

     - expression中使用了test命令中的采用的标准字符串比较 以及可以进行test命令未提供的另一个特性——**模式匹配**

     ```shell
     [[ expression ]]
     
     # 模式匹配 r*
     if [[ $USER == r* ]]
     then 
     	...
     fi
     ```

     

3. `for`

   - for循环假定每个值都是用空格分隔 

   - 如果在单独的数据值中包含空格 则需要用空格将该数据值包含起来

     - 在for中使用双引号时 shell并不会将双引号当成值的一部分

     ```shell
     for s in This is "New Mexico"
     do
     	..
     done
     ```

     

   - 更改字段分隔符

     - 特殊的环境变量 `IFS`
     - **在脚本中修改IFS使其能正确输出带有空格或制表符的数据**

     ```shell
     # 告诉Shell脚本在数据值中忽略空格以及制表符 
     IFS=$'\n' 
     ```

     

   ```shell
   for var in item1 item2 ... itemN
   do
       command1
       command2
       ...
       commandN
   done
   ```

   - 写成一行

   ```shell
   for var in item1 item2 ... itemN; do command1; command2… done;
   ```

   - 顺序输出字符串中的字符

   ```shell
   #!/bin/bash
   
   # 不写成 "for str in This is a string"
   for str in This is a string
   do
       echo $str
   done
   ```

   

4. while

   ```shell
   i=1
   while (($i <= 5))
   do
           echo $i
           let "i++" # 相当于(())赋值表达式
   done
   
   # 输出
   1
   2
   3
   4
   5
   ```

   ```shell
   echo "Press ctrl-D to exit"
   echo -e "input you favorite website name\n"
   while read FLIM
   do
           echo "yes $FLIM is a good website"
   done
   ```

   

5. **() (()) [] [[]]**

   - 使用`()`
     - **命令组** 括号中的命令将会新开一个子shell顺序执行，所以括号中的变量不能够被脚本余下的部分使用。括号中多个命令之间用分号隔开，最后一个命令可以没有分号，各命令和括号之间不必有空格。
     
       ```shell
       file="dirctory"
       for state in $(car $file)
       ```
     
       
     - **命令替换** 等同于`cmd`，shell扫描一遍命令行，发现了$(cmd)结构，便将$(cmd)中的cmd执行一次，得到其标准输出，再将此输出放到原来命令。有些shell不支持，如tcsh。
     - **用于初始化数组** 如：array=(a b c d)
     
   - 使用`(())`时解释器应该为bash 否则提示未找到变量等错误信息

   ```shell
   #!/bin/bash
   a=10
   if (($a > 10))
   then 
   	echo "a > 10"
   else
   	echo "a<=10"
   fi
   
   # 输入命令 bash if.sh
   # 输入 sh if.sh会出错
   ```

   - 使用`[]`执行基本的算数运算，同时也是与test命令同义的特殊bash命令

   - 使用`[[]]`用于条件判断结构

     - &&、||、<和> 操作符能够正常存在于[[ ]]条件判断结构中，但是如果出现在[ ]结构中的话，会报错。

     ```shell
     if [ $a -ne 1] && [ $a != 2 ]   
     if [[ $a != 1 && $a != 2]]
     ```

     

6. `case`

   ```shell
   case variable in 
   pattern1) 
   	commands1;;
   pattern2)
   	commands2;;
   *)
   	default_commands;;
   esac
   ```

   

# 重定向输入和输出

1. **输出重定向** ：将命令的内容输入到对应的文件
   - `>`    覆盖源文件内容 
   - `>>`  在源文件上追加内容
2. **输入重定向** 
   - `<`   将文件的内容重定向到命令
   - `<<`  **内联输入重定向** 无需使用文件进行重定向 只需要在命令行中指定用于输入重定向的数据即可
   - 
