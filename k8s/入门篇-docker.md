# K8s入门篇-Docker

## 安装 指南

```shell
sudo apt update 
sudo apt install -y docker.io 

sudo usermod -aG docker ${USER}
# 更新docker用户组
newgrp docker
```

## 安装vmware-tools

[(69条消息) 【VMware】 VMware Tools安装步骤（windows10）_Alix_sz的博客-CSDN博客_vmtools安装](https://blog.csdn.net/humanof/article/details/127494561)

## 容器与外界互通

1. 使用数据卷实现容器挂载
2. Docker 提供了三种网络模式，分别是 null、host 和 bridge

## 常见错误指南

1. 关于指令生成层的问题：只有 RUN, COPY, ADD 会生成新的镜像层，其它指令只会产生临时层，不影响构建大小，官网的镜像构建最佳实践里面有提及，https://docs.docker.com/develop/develop-images/dockerfile_best-practices/ 
2. docker pull拉取慢：在[容器镜像服务 (aliyun.com)](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)查找镜像源  [(69条消息) Docker更改镜像源_普通网友的博客-CSDN博客_修改docker镜像源](https://blog.csdn.net/segegefe/article/details/126327589)
   - "registry-mirrors": ["https://9q1nmamk.mirror.aliyuncs.com"]
3. 端口号映射需要使用 bridge 模式，并且在 docker run 启动容器时使用 -p 参数，形式和共享目录的 -v 参数很类似，用 : 分隔本机端口和容器端口
4. Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon：在/etc/docker/daemon.json下的配置中某些字段出错导致linux中docker服务没有成功重启

## Namespace/Cgroups/Rootfs

### NameSpace

1. 容器本质上是一个运行在宿主机上的进程，使用系统函数clone来对应用做一个假象
2. 容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是同一个宿主机的操作系统内核。  

### Cgroups

1. Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。  
2. Cgroups 还能够对进程进行优先级设置、审计，以及将进程挂起和恢复等操作。  
3. Linux Cgroups 的设计还是比较易用的，简单粗暴地理解呢，它就是一个子系统目录加上一组资源限制文件的组合。而对于 Docker 等 Linux 容器项目来说，它们只需要在每个子系统下面，为每个容器创建一个控制组（即创建一个新目录），然后在启动容器进程之后，把这个进程的 PID 填写到对应控制组的 tasks 文件中就可以了。  