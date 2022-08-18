# Docker命令

1. **镜像命令概述**

   ```shell
   # 镜像命令
   
   # 列出宿主机上所有的镜像
   docker images
   # 各列含义
   # REPOSITORY  ：镜像的仓库源
   # IMAGE ID    : 镜像的ID
   # TAG         ：镜像的标签
   # CREATED 	  ：镜像创建的时间
   # SIZE        ：镜像大小
   
   # 在容器中运行镜像
   docker run [参数] [REPOSITORY]:[TAG]/[IMAGE ID] [COMMAND]
   # 参数
   -d : 容器在宿主机以后台方式运行
   -i : 进行交互式操作
   -t : 以伪终端tty方式运行
   -p : 指定容器开放给外部的端口号
   --name : 给容器命名
   --restrart : always->容器退出时自动重启容器 / 
   			on-failure->当容器退出代码非0时才会自动重启
   # docker run --name my-redis -d redis
   # 在容器中运行镜像ubuntu标签为18.04 并通过交互式的SHELL运行
   # docker run -i -t ubuntu:18.04 /bin/bash
   
   # 在仓库获取新的镜像
   docker pull [REPOSITORY]:[TAG]
   
   # 查找镜像
   # 可以从本地仓库或Docker Hub(公有仓库)查找镜像
   docker search [REPOSITORY]
   
   # 删除镜像
   docker rmi [REPOSITORY]
   
   # 创建镜像
   # 在原有的镜像源上进行修改然后以此再创建新的镜像
   # 方法：通过dockerfile/docker build或者docker commit
   docker commit [OPTIONS] [源REPOSITORY]/[IMAGE ID] [目的REPOSITORY]:[TAG]
   # 参数
   -m : 信息注释
   -a : 指定镜像作者
   -c : 对新创建的镜像的Dockerfile应用其新添加的命令
   -p : 提交过程中暂停源容器运行
   # docker commit -m 'ubuntu1->ubuntu2' -a 'zyb' ubuntu1 ubuntu2:v2
   # docker commit -c 'CMD [/bin/bash]'  -a 'zyb'  ubuntu2 ubuntu3:v3
   
   # 设置镜像标签
   docker tag [REPOSITORY]/[IMAGE ID] [REPOSITORY]:[TAG]
   # docker tag 860c279d2fec zyb/ubuntu:v1
   
   
   
   ```

   

2.  **基础命令**

   ```shell
   # 查看宿主机所有容器和镜像
   docker info
   
   # 查看正在运行的容器
   docker ps 
   -a : 查看当前系统中所有容器的列表（包括当前不在运行的容器
   -l : 列出最后一次运行的容器
   
   # 重启容器
   # 当退出容器时 若需要重新启动容器
   # 容器重新启动时 会沿用之前docker run指定的参数来运行
   docker start   [container]
   docker restart [container]
   
   # 附着到容器上
   # 回到容器的Bash提示符
   # 如果退出容器的Shell终端 容器也会随之停止运行
   docker attach [container]
   
   # 获取容器日志
   docker logs
   # 参数
   -t : 为每条日志项加上时间戳
   -f : 监控Docker日志
   --tail string : 获取日志的最后string行内容
   
   # 查看容器内部正在运行的进程
   docker top [container]
   
   # 在容器内部额外启动新进程
   docker exec [OPTIONS] [container] [COMMAND]
   # 参数
   -d : 运行一个后台进程 指定的是在内部执行COMMAND的容器名以及要执行的COMMAND
   -t : 为执行的进程创建tty
   -i : 为进程捕捉STDIN
   # docker exec -d ubuntu1 touch /etc/newfile.txt
   
   # 停止守护容器
   docker stop [container]
   docker kill [container]
   
   # 深入了解容器信息
   docker inspect [container]
   -f : 指定需要的查询结果
   # 指定查询容器当前的运行状态
   # docker inspect -f='{{ .State.Runing}}' ubuntu1 
   
   # 删除容器
   # 无法删除运行中的容器
   docker rm [container]
   -f : 强制删除当前运行中的容器
   ```

   