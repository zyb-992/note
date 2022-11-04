# 文件与目录管理

### 目录与路径

1. `cd`

   ```shell
   cd /a/b
   # 执行cd 直接回到用户主目录
   cd
   
   cd /c/d
   # 回到执行cd /c/d时的工作目录
   cd - 
   ```

   

2. `mkdir`

   ```shell
   -m 为新建目录添加ugo权限
   -p 递归创建目录
   ```



3. `rmdir` 删除空目录



### 文件内容查阅

| 命令 | 内容                             |
| ---- | -------------------------------- |
| cat  | 从文件第一行开始显示文件所有内容 |
| tac  | 倒序显示文件所有内容             |
| nl   | 显示时设置输出行号的格式         |
| more | 一页一页的显示文件内容           |
| less | 同more类似 但是可以往前翻页      |
| head | 只看文件头几行                   |
| tail | 只看文件末尾几行                 |
| od   | 以二进制的方式读取文件内容       |



1. `more`

   ![image-20220927100529118](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209271005154.png)

   

2. `less`

   ![image-20220927100626968](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209271006006.png)



3. `touch`及文件的三个主要变动时间

   ![image-20220927100800249](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209271008292.png)

   - touch

     ![image-20220927101415015](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209271014052.png)



### 文件与目录的默认权限与隐藏权限

1. `umask` : 文件预设权限，指定用户在建立文件或目录时的权限默认值

2. **umask代表的是该默认值需要被减掉的权限**

   ```shell
   zyb@ubuntu:~$ umask
   0002
   
   zyb@ubuntu:~$ umask -S
   # 代表u-0 g-0 o-2 即775
   u=rwx,g=rwx,o=rx
   ```

3. 对于文件来说 不需要可执行权限 对于目录来说 需要可执行权限



### 指令与文件搜寻

1. 脚本文件名的搜寻

   - `which [-a] command` : 根据PATH这个环境变量所规范的路径中去寻找所执行命令的绝对路径

   ![image-20220927110722283](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209271107339.png)

   ​	

2. 文件档名的搜寻

   - `whereis`

   - `locate`/`upgradedb`

   - `find`

     ![image-20220927110954587](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209271109642.png)

![image-20220927111613647](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209271116714.png)