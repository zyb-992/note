# 第三章

### 浏览文件系统

1. `cd`

   - 从任何一级中切换回用户主目录

   ```shell
   gec@ubuntu:/etc$ cd 
   gec@ubuntu:~$ 
   ```

   

2. `ls`

   | 参数 | 作用                           |
   | ---- | ------------------------------ |
   | -F   | 区分文件还是目录               |
   | -a   | 显示隐藏文件(以.为前缀的)      |
   | -R   | 当前目录下包含的子目录中的文件 |
   
   - 使用正则表达式

   ```shell
   # 查找my_scrapt以及my_script文件
   
      ls -l my_scr[ai]pt
   
      # 查找该索引下包含a\b\c--->到i的文件
   
      ls -l my_scr[a-i]pt
   
      # 查找除了my_scrrpt之外的文件
   
      ls -l my_scr[!r]pt
   ```

   ![image-20220916161132325](https://raw.githubusercontent.com/zyb-992/Photobed/master/zyb/202209161611352.png)

3. `touch`

   - 用于创建文件
   - 改变文件的修改时间

### 链接文件

> Linux下的文件系统

1. 符号链接(软链接)

   ```shell
   # file2 为 创建的file1的软链接
   ln -s file1 file2

   - 软链接在源文件被删除后就会消失 查询不到其中的内容
   - 软链接与源文件指向的innode编号不同

2. 硬链接

   ```shell
   # file2 为 创建的file1的硬链接
   ln file1 file2
   ```

   - 硬链接在源文件被删除后还保存着并查询得到源文件中的内容 
   - 软链接与源文件指向的innode编号相同

### 重命名文件

1. `mv`

   - 可以将文件和目录移动到另一个位置或者重新命名
   - 移动过后源文件或者源目录的大小和时间戳都没有改变 
   - 改变的只有绝对位置或者名称

   ```shell
   gec@ubuntu:~/zyb$ ls
   1.txt
   
   gec@ubuntu:~/zyb$ mv 1.txt 2.txt
   
   gec@ubuntu:~/zyb$ ls
   2.txt
   ```

   

### 删除文件/目录

1. `rm`

   | 参数 | 作用 |
   | ---- | ---- |
   | -r   |      |
   | -R   |      |
   | -f   |      |

   

2. `rmdir`

   

### 查看文件内容

- `file`

- `cat`

  - -n  显示行号

  - -b  给有文本的行添加行号

  - -T  不让文本中制表符起作用

- `more`

- `less`

- `tail`：显示文件末尾n行

- `head` ：显示文件开头n行