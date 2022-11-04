# shell间的父子关系

### 认识与学习Bash

1. 什么是shell?

   1. Shell俗称壳(用来区别于核)，是指“为使用者提供操作界面”的软件（命令解析器）。它类似于DOS下的command.com和后来的cmd.xe；它接收用户命令，然后调用相应的应用程序
   2. 透过shell将我们输入的指令与Linux内核沟通 让内核可以控制硬件来正确无误的工作
   3. /bin/bash是Linux预设的shell，兼容于sh，并且依据一些使用者需求而加强的shell版本
   4. bash的主要优点
      - 记忆使用过的指令：在用户主目录`~`下的`.bash_history`可以查询到我们曾经下达过的指令；它记录的是前一次登入以前所执行过的命令，而此次登入执行的指令都被暂存在内存中，当关机或者注销系统后，此次的指令记忆才会记录到`.bash_history`中
      - 命令与文件补全功能(`tab`)
      - 命令别名设定功能(`alias`)
      - 通配符

   ![image-20220917161512880](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209171615006.png)

2. 查询指令是否为内建指令：type 

```

```

3. 指令下达与快速编辑按钮

   - 使用反斜杠`\`来转义[Enter]

     ```shell
     gec@ubuntu:~$ cp file1 \
     > dir1
     ```

   - 使用组合键

     | 组合键 | 功能                         |
     | ------ | ---------------------------- |
     | ctrl+u | 从前删除指令串               |
     | ctrl+k | 从后删除指令串               |
     | ctrl+a | 光标移动到最前               |
     | ctrl+e | 光标移动到最后               |
     | ctrl+l | 清屏（当前正在输入的命令除外 |

     

### SHELL的变量功能

1. 变量的设定功能

   ![image-20220927111942955](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209271119013.png)

   - `unset` : 使用unset取消(删除)自定义变量

   

2. **环境变量**

   1. 使用`env`观察环境变量

      ```shell
      # 列出当前linux的所有环境变量
      env
      ```

      

   2. 使用`set`观察环境变量与所有自定义变量

      ```shell
      # 列出环境变量与所有自定义变量
      set
      ```

      

   3. 在Linux预设中一般设置大写字母来设定的变量都是系统内定需要的变量

   | 变量名 | 内容                                                         |
   | ------ | ------------------------------------------------------------ |
   | PS1    | 命令提示字符，如在伪终端输入命令Enter后 命令执行完后会再次出现zyb@ubuntu:~$ 这一字段 |
   | $      | 当前shell的线程代号（PID                                     |
   | ？     | 上个执行指令的返回值（成功返回0 失败等返回127                |

   ![](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209271132568.png)

   

   4. `export` : 将自定义变量转成环境变量

      

### **读取键盘输入参数**

1. `read` 

   ![image-20220928103541031](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220928103541031.png)

2. `declare`

   ![image-20220928103748087](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209281037194.png)

   ```shell
   # 变量类型默认为字符串 若不指定变量类型 则sum=100+30+50默认sum为字符串
   zyb@ubuntu:~$ echo $sum
   100+30+50
   zyb@ubuntu:~$ declare -i sum=100+30+50
   
   zyb@ubuntu:~$ echo $sum
   180
   ```

   

### 与文件系统及程序的限制关系

1. 用户登录时的bash是可以限制用户使用的某些系统资源的，如可以开启的文件数量，可以使用的CPU时间，可以使用的内存总量等

2. `ulimit`

   ![image-20220928110330922](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209281103007.png)



3. 自定义变量或环境变量中内容的删除、取代或替换

   ![image-20220928111743140](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209281117194.png)





### 历史命令查询

1. `history`

   ```shell
   # 查询执行过的3条命令
   history 3
   
   # 查询保存在~/.bash_history所有的命令集合 
   history
   ```

   ![image-20220928135135409](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209281351487.png)

![image-20220928140527758](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209281405797.png)