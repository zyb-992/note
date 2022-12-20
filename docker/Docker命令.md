# Docker命令

> 书籍 [数据卷 - Docker — 从入门到实践 (gitbook.io)](https://yeasy.gitbook.io/docker_practice/data_management/volume)

## **镜像命令概述**

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

# 查找镜像
# 可以从本地仓库或Docker Hub(公有仓库)查找镜像
docker search [REPOSITORY]

# 删除镜像
docker rmi [REPOSITORY]

# 在仓库获取新的镜像
docker pull [REPOSITORY]:[TAG]

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



## **容器基础命令**

```shell
# 查看宿主机所有容器和镜像
docker info

# 在容器中运行镜像
docker run [参数] [REPOSITORY]:[TAG]/[IMAGE ID] [COMMAND]
# 参数
-d : 容器在宿主机以后台方式运行
-w : 设置容器工作文件夹
-i : 对容器进行交互式操作
-t : 以伪终端tty方式运行
-P : Docker会随机映射一个host端口到内部容器开放的网络端口
-p : 指定容器开放给外部的端口号, 
	 支持的格式：ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort
--name : 给容器命名
--restrart : 
		- always 	 -> 容器退出时自动重启容器
		- on-failure -> 当容器退出代码非0时才会自动重启
--volume-from : 指定挂载的父容器
# 在容器中运行镜像ubuntu标签为18.04 并通过交互式的SHELL运行
# docker run -i -t ubuntu:18.04 /bin/bash

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
# CTRL+T+Q 回到宿主机
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

# 在容器内部额外启动新的镜像进程
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

- `docker diff`

  ```shell
  # 检查对容器文件系统上的文件或目录的更改
  docker diff CONTAINER
  # 检查下列这几种类型
  - A : A file or directory was added
  - D : A file or directory was deleted
  - C : A file or directory was changed
  ```

## Vloume

- 数据卷：可供多个容器使用的特殊目录
  - 特点
    - 可在容器之间共享和重用
    - 对数据卷的修改会立马生效
    - 对数据卷的更新不会影响镜像
    - 数据卷默认会一直存在，即时容器被删除
  - 

```shell
# 创建容器卷
docker volume create v_name

# 查看容器卷
docker volume ls v_name

# 检查某容器卷相关信息
docker volume inspect v_name

# 删除容器卷
docker volume rm v_name


#  Remove all unused local volumes
docker volume prune
```

- 数据卷加载到容器中

  ```shell
  # 使用--mount 加载my-vol数据卷到容器的/usr/share/nginx/html目录中
  docker run  -d -p \
  # -v my-vol:/usr/share/nginx/html  \
  --mount source=my-vol, target=/usr/share/nginx/html \
  nginx:alpine
  ```

- 挂载主机目录作为数据卷到容器中

  - **本地目录的路径必须是绝对路径**

  ```shell
  # 加载主机的 /src/webapp 目录到容器的/usr/share/nginx/html目录。
  docker run -d -p \
  --mount type=bind, source=/src/webapp, target=/usr/share/nginx/html \
  nginx:alpine
  ```

- Docker挂载主机目录的默认权限是**读写**，用户可以增加`readonly`设置为只读

  ```shell
  $ docker run -d -P \
  # -v /src/webapp:/usr/share/nginx/html:ro \
  --mount type=bind,source=/src/webapp,target=/usr/share/nginx/html,readonly \
  nginx:alpine
  ```

  

## 挂载点

## 每个容器只运行一个应用

> [Multi container apps | Docker Documentation](https://docs.docker.com/get-started/07_multi_container/)
>
> Part 7 : Mutli-container apps：**each container should do one thing and do it well**
>
> - There’s a good chance you’d have to scale APIs and front-ends differently than databases
> - Separate containers let you version and update versions in isolation
> - While you may use a container for the database locally, you may want to use a managed service for the database in production. You don’t want to ship your database engine with your app then.
> - Running multiple processes will require a process manager (the container only starts one process), which adds complexity to container startup/shutdown

## 两个容器期间如何通信

> [Multi container apps | Docker Documentation](https://docs.docker.com/get-started/07_multi_container/)
>
> Part 7 : Mutli-container apps Container networking：
>
> how do we allow one container to talk to another? The answer is **networking**

```shell
# 创建网络
docker network create 

docker network ls 

docker network inspect

docker network connect

docker network disconnect

docker network prune
# OPTIONS 
# --filter
# 		utils=5m
# 		label=<key>
# --force

docker network rm
```

- `docker network create `

  | Name, shorthand   | Default  | Description                                             |
  | ----------------- | -------- | ------------------------------------------------------- |
  | `--attachable`    |          | Enable manual container attachment                      |
  | `--aux-address`   |          | Auxiliary IPv4 or IPv6 addresses used by Network driver |
  | `--config-from`   |          | The network from which to copy the configuration        |
  | `--config-only`   |          | Create a configuration only network                     |
  | `--driver` , `-d` | `bridge` | Driver to manage the Network (bridge, overlay)          |
  | `--gateway`       |          | IPv4 or IPv6 Gateway for the master subnet              |
  | `--ingress`       |          | Create swarm routing-mesh network                       |
  | `--internal`      |          | Restrict external access to the network                 |
  | `--ip-range`      |          | Allocate container ip from a sub-range                  |
  | `--ipam-driver`   |          | IP Address Management Driver                            |
  | `--ipam-opt`      |          | Set IPAM driver specific options                        |
  | `--ipv6`          |          | Enable IPv6 networking                                  |
  | `--label`         |          | Set metadata on a network                               |
  | `--opt` , `-o`    |          | Set driver specific options                             |
  | `--scope`         |          | Control the network's scope                             |
  | `--subnet`        |          | Subnet in CIDR format that represents a network segment |

- 
