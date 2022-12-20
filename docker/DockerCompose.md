# Docker Compose

> Compose 负责实现对 Docker 容器集群的快速编排
>
> `Compose` 允许用户通过一个单独的 `docker-compose.yml` 模板文件（YAML 格式）来定义一组相关联的应用容器为一个项目。
>
> Network Namespace:[Introducing Linux Network Namespaces - Scott's Weblog - The weblog of an IT pro focusing on cloud computing, Kubernetes, Linux, containers, and networking (scottlowe.org)](https://blog.scottlowe.org/2013/09/04/introducing-linux-network-namespaces/)
>
> Docker-从入门到实践：[Compose 模板文件 - Docker — 从入门到实践 (gitbook.io)](https://yeasy.gitbook.io/docker_practice/compose/compose_file)

## 重要概念

- 服务：一个应用容器，实际上可以运行多个相同镜像的实例。
- 项目：由一组关联的应用容器组成的一个完整业务单元。
- 使用

- `Compose` 的默认管理对象是项目，通过子命令对项目中的一组容器进行便捷地生命周期管理。
- Compose实现上调用了 Docker 服务提供的 API 来对容器进行管理。

- Copomse文件中的volume是可读写的 而config和secret是只读的
- Compose文件的默认路径是工作目录中的compose.yaml（首选）或者compose.yml
- 