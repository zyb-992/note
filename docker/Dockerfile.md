# Dockerfile

1. 执行docker build命令时 Dockerfile中所有指令都会被执行并且提交 在该命令执行成功后返回一个新的镜像

2. FROM

   ```shell
   # 为后续指令设置基础镜像
   FROM [--platform=<platform>] <image>[:<tag>] [AS <name>] 
   # 用于指定一个容器启动时要运行的命令
   # docker run命令会覆盖CMD指令 
   # 若Dockerfile中指定了CMD指令 同时docker run命令也指定了要运行的命令 命令行中指定的命令会覆盖Dcokerfile中的CMD指令
   
   # FROM指令支持之前出现的指令生命的变量
   ARG CODE_VERSION=latest
   FROM base:${CODE_VERSION} 
   # -> FROM base:latest
   
   ```

3. CMD：创建容器时会执行的命令
   1. CMD在Dockerfile中只允许有一条 若有多条则按最后一条的CMD命令执行
   2. CMD主要用途是在最后运行的容器中执行命令

```shell
# 三种形式
# exec form
CMD ["executable", "param_1", "param_2"] 
# 作为默认参数传递给ENTRYPOINT
CMD ["param_1", "param_2"]
# shell form
CMD command param_1 param_2
# CMD [ "bin/bash" ]
# sudo docker run -i -t zyb/test 
# 相当于 sudo docker run -i -t zyb/test bin/bash
# -> root@aseauiewb24:/#
```

4. RUN：用于在当前镜像中执行指令

   ```shell
   # 两种形式
   # shell form
   RUN <COMMAND>
   # exec form
   RUN ["executable", "param_1", "param_2"] 
   
   # 两种情况等价 
   RUN /bin/bash -c 'source $HOME/.bashrc; \ 
   echo $HOME'
   
   RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'
   
   # exec form
   # 以JSON格式解析 所以传递需要加双引号 不是单引号
   RUN ["bin/bash", "-c", "echo hello"]
   
   # 在exec form中 必须对反斜杠进行转义
   `RUN ["c:\windows\system32\tasklist.exe"] 不对 需要转义`
   RUN ["c:\\windows\\system32\\tasklist.exe"]
   
   # --no-cache
   # 指令缓存在下一次构建期间重用 可以使用标志--no-cache使其在下次构建镜像时失效
   ```

   

5. EXPOSE：告诉Docker该容器运行时需要监听哪些指定的网络端口

   1. 只是暴露了运行该镜像时的容器的端口
   2. 在docker run时还是 需要指定-p选项来指定端口绑定

   ```shell
   # 格式
   EXPOSE <port> [<port>/<protocol>...]
   
   EXPOSE 80/udp
   EXPOSE 80/tcp
   ```

   

6. LABEL：将元数据添加到镜像中

   ```shell
   # 格式 
   LABEL <key>=<value> <key>=<value> <key>=<value> ...
   
   # 例子
   LABEL "com.example.vendor"="ACME Incorporated"
   LABEL com.example.label-with-value="foo"
   LABEL version="1.0"
   # 值需要包括空格时使用反斜杠 \ 来转义
   LABEL description="This text illustrates \
   that label-values can span multiple lines."
   
   # 在一行LABEL上添加多个标签
   LABEL multi.label1="value1" \
         multi.label2="value2" \
         other="value3"
         
   # 查看当前镜像的标签 -> 仅仅显示标签
   docker image inspect --format='' myimage
   ```

   

7. ENV： 将环境变量设置为值 将出现在构建阶段的所有后续指令的环境中 在其他指令中以内联方式替换

   1. **使用ENV生成的环境变量最终会添加到最终生成的镜像中 因此当通过该镜像运行容器时 该环境变量也会起作用**

```shell
# 格式
ENV <key>=<value>

# 例子
ENV MY_NAME="John Doe"
ENV MY_DOG=Rex\ The\ Dog
ENV MY_CAT=fluffy

ENV MY_NAME="John Doe" MY_DOG=Rex\ The\ Dog \
    MY_CAT=fluffy
    
# 为单个命令设置环境变量
# 在此命令设置后下个命令执行后新生成的镜像内就不会有之前定义的环境变量
ARG DEBIAN_FRONTEND=noninteractive
```

8. ENTRYPOINT：容器入口点

```shell
# 2 forms
# exec form
ENTRYPOINT ["executable", "param_1", "param_2"]
# 外壳形式
ENTRYPOINT command param1 param2


# 通过传递空字符串重置容器入口点
docker run -it --entrypoing="" mysql bash

# 例子
# 编写Dockerfile
FROM redis:latest
ENTRYPOINT ["redis-cli","-p","6379"]

# 构建新镜像
docker build -f dockerfile_test -t new_redis:v1 .
# 容器运行
docker run --entrypoint="redis-server" new_redis:v1
# 结果是运行了redis服务器
```

**CMD和ENTRYPOINT的区别**

- CMD所执行的命令若docker run后面的COMMAND存在则会被覆盖 而ENTRYPOINT则不会 它会追加到ENTRYPOINT的参数上

  ```shell
  CMD ["ls",'-a']
  ENTRYPOINT ["ls","-a"]
  
  docker run -it ubuntu1 -l
  -> 对于CMD来说 会报错: docker run -it ubuntu1 -l
  -> 对于ENTRYPOINT来说: docker run -it ubuntu1 ls -a -l
  ```

  

- ![image-20220805185410334](C:\Users\zyb\AppData\Roaming\Typora\typora-user-images\image-20220805185410334.png)

8. ADD：用来将构建环境下的文件和目录复制到镜像中

   1. 源文件位置：指向源文件的位置参数可以是URL或者构建上下文中的文件名或目录 **不能对构建目录或者上下文之外的文件进行ADD操作**
   2. 目的文件位置：Docker通过目的地址参数末尾的字符来判断文件源是目录还是文件 如果目的地址以**/**结尾 那么Docker认为源位置指向的是目录 若不是 则认为源位置指向的是文件
   3. 文件源可以使用URL的格式
   4. **路径必须位于构建上下文中 因为Dockerfile第一步是将上下文目录和子目录发送到Docker守护程序中**
   5. **ADD处理本地归档文件时（tar,archive) 若将归档文件指定为源文件 Docker会自动将归档文件解开(解压缩) 然后添加到目的文件位置中 从远程的URL中的资源则不会被解压缩** COPY命令则不会
   6. 如果目的位置不存在的话 Docker会自动为我们创建这个路径 包括路径中的任何目录 新创建的文件和目录的模式为0755 且UID和GID都是0
   7. **ADD指令会使构建缓存变得无效**

   ```shell
   # 格式
   ADD [--chown=<user>:<group>] <src>... <dest>
   ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]
   
   # Adding a git repository
   ADD [--keep-git-dir=<boolean>]  <git ref> <dir>
   # 允许直接添加一个git仓库到镜像中
   ADD --keep-git-dir=true https://github.com/moby/buildkit.git#v0.10.1 /buildkit
   
   ```

9. COPY：COPY在构建上下文中复制本地文件 而不会去做文件提取和解压的工作

   1. 文件源路径必须是一个与当前构建环境相对的文件或目录 本地文件都放到和Dockerfile同一个目录下 不能复制该目录之外的任何文件 因为构建环境将会上传到Docker守护进程中 而复制是在Docker守护进程中进行的 任何位于构建环境之外的东西都是不可用的 
   2. COPY指令的目的位置必须是容器内部的一个绝对路径

   ```shell
   COPY <src> <dest>
   
   # 同ADD命令的文件规则
   COPY hom* /mydir/
   COPY home?.txt /mydir/
   ```

   

10. 