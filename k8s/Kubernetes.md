# Kubernetes

##  架构

## Pod 

> 将pod调度到指定Node上运行 [K8S 将 pod 调度到指定 nodes 上运行 | 海牛部落 高品质的 大数据技术社区 (hainiubl.com)](http://www.hainiubl.com/topics/75623)

### 了解Pod——Kuberntes操作资源的一个对象

1. Pod实际上是一个抽象的概念，它是一台逻辑上的主机，并不真实存在，运行在同一个Pod中的进程与运行在物理机或虚拟机上的进程类似，只是每个进程（包含支持进程）都封装在同一个容器之中，Pod运行在一个具体的工作节点上
2. 同一个Pod内的所有容器共享相同的Linux命名空间，一个Pod中的所有容器都在相同的Network和UTS 命名空间下运行，因此共享相同的主机名和网络接口，同时也能共享相同的PID命名空间

### 同个Pod中的容器如何共享相同的IP和端口空间

1. 同个Pod中的容器运行于相同的Network命名空间中，它们共享相同的IP地址和端口空间，即同个Pod中的的多个容器需要注意不能绑定到相同的端口号，否则会发生西端口冲突，不同Pod中之间不会遇到端口冲突
2. 一个Pod中的所有容器具有相同的loopback网络接口，因此容器可以通过localhost与同个Pod中的其他容器进行通信

### Pod之间的通信

1. Kubernetes集群中的所有pod都在同一个共享网络地址空间中，每个Pod都可以通过其他Pod的IP地址来实现相互访问，不需要进行NAT（网络地址转换）
2. 不论不同Pod之间是被安排在同一个或者不同的工作节点上，这些Pod内的容器都能想在无NAT的平坦网络中一样相互通信，每个Pod都有自己的IP地址，并且通过专门的网络实现Pod之间的相互访问，**这个专门的网络通常是由额外的软件基于真实链路实现**

### 通过Pod合理管理容器

#### 	将多层应用分散到多个Pod中

1. 基于计算资源考虑，
2. 基于扩缩容考虑，Pod是Kubernetes扩缩容的基本单位，而不是容器，它不能横向扩缩单个容器，只能扩缩整个Pod，对于`kubectl scale`来说，更倾向于分别将多层应用分散到多个Pod中，以此来实现高可用

#### 	何时在Pod中使用多个容器

1. 将多个容器添加到单个 Pod 的主要原因是应用可能由一个主进程和一个或多个辅助进程组成

2. 回顾一下容器应该如何分组到 pod 中:`当决定是将两个容器放入一个 pod 还是两个单独的 pod 时，我们需要问自己以下问题 `

   - 它们需要一起运行还是可以在不同的主机上运行?
   - 它们代表的是一个整体还是相互独立的组件?
   - 它们必须一起进行扩缩容还是可以分别进行?

3. 基本上 我们总是倾向于在单独的Pod中运行容器，除非有特定原因要求它们是同一Pod的一部分

   <img src="D:\Program Files\电子书\go\md\图片\image-20230219134040494.png" style="zoom:67%;" />

   <img src="D:\Program Files\电子书\go\md\图片\image-20230219134238971.png" style="zoom:67%;" />

### 通过Yaml或Json创建Pod

- `kubectl create`与`kubectl apply`区别
  - 
- 

### 查看在Pod中运行的应用程序的日志

1. 不用进入到Pod的容器中，只需要执行`kubectl logs pod_name`即可

2. 若需要获取多容器Pod的日志时至指定容器名称，只需要在命令中指定`-c`，即`kubectl logs pod_name -c <容器名称>`

3. 当一个Pod被自动/手动删除时，它的日志也会被删除，如果希望在 Pod被删除之后仍然可以获取其日志，我们需要设置中心化的、集群范围的日志系统，将所有日志存储到中心存储中。

   ![](D:\Program Files\电子书\go\md\图片\image-20230219142721474.png)

### 使用标签组织Pod

1. 标签能够基于任意标准将集群中的Pod组织成更小群体的形式，使得处理系统的每个开发人员和系统管理员能够轻松地看到集群中各个Pod的真正含义
2. **标签(Label)**是可以附加到资源的任意键值对，在资源中键是唯一的，**标签通过标签选择器来选择具有该确切标签的资源**
3. 一个资源可以拥有多个标签，也可以在现有基础上添加标签或修改现有标签而无需重新创建资源
4. 查看标签
   - `kubectl get pods --show-labels`
   - `kubectl get pods -L label_name1,label_name2`
5. 修改现有Pod的标签：`仅在Pod运行时有效，删除后重新创建该键值对会消失或保持原样`
   - `kubectl label pod pod_name key=value`
   - 更改现有标签：`kubectl label pod pod_name key=value --overwrite`

### 通过标签选择器列出pod子集：标签要与标签选择器结合

1. 标签选择器允许我们选择标记有特定标签的Pod子集，并对这些Pod执行操作，它是一种能够根据是否包含具有特定值的特定标签来过滤资源的准则

2. 使用`kubectl get pod pod_name -l label_name`来列出Pod

   ```shell
   # 列出带有env标签的pod
   kubectl get pod -l env
   
   # 列出没有env标签的pod
   kubectl get pod -l '!env'
   
   # 
   kubectl get pod -l env=zyb
   kubectl get pod -l env!=zyb
   kubectl get pod -l env in (zub, zyb)
   kubectl get pod -l env notin (zub, zyb)
   ```



## ReplicationController / ReplicaSet

### 存活探针——保持Pod的健康运行

1. Kubernetes可以通过存活探针来检查容器是否还在运行，可为pod中的每个容器单独指定存活探针，若探测失败，则k8s将定期执行探针并重新启动容器
2. 三种**探测容器机制**
   - HTTP GET探针对容器的IP地址（指定端口与路径）执行HTTP GET请求
   - TCP套接字探针尝试与容器指定端口建立TCP连接：建立连接则探测成功 否则失败
   - Exec探针在容器内执行任意命令：在容器内执行任意命令，检查命令退出状态码，若0则探测成功，其他状态码默认失败

3. 创建基于HTTP的存活探针

   在pod的yaml文件中指定**`livenessProbe`**，在端口8080路径上执行HTTP GET请求

   <img src="D:\Program Files\电子书\go\md\图片\image-20230219234302529.png" style="zoom:50%;" />

4. 当容器崩溃重启时，使用`kubectl describe`来查看具体细节，`Events`会告知容器为什么终止运行，同时其中的`Liveness`字段显示了存活探针的附加信息，

   - delay表示在容器启动后立即执行存活探测
   - timeout表示容器必须在这个时间段内对探针的请求进行回应，否则这次探测视为失败
   - period表示存活探针每隔10秒查看容器是否正常运行
   - failure表示在连续探测几次失败后视为容器终止

   <img src="D:\Program Files\电子书\go\md\图片\image-20230219235223771.png" style="zoom:67%;" />

5. 定义探针时，可以在yaml文件中初始化上述附加参数，例如`livenessProbe.initialDelaySeconds`来设置初始延迟delay

### ReplicationController

>  使用存活探针来检测容器的运行状态，能够保证容器在该Pod中异常终止时能够自动重启，通过该Pod所在节点的Kubelet组件来执行，但若是自身节点崩溃导致容器无法自动重启，Kubelet就执行不了这个操作，因此为了确保Pod中容器每时每刻的正常运行，需要使用Kubernetes中的`ReplicationController`对象等机制来实现对Pod的监控，使其在节点崩溃时能够将该节点上的Pod迁移到其他正常节点上运行

1. 假设某个Pod由ReplicationController监管，当它因节点某些原因异常退出后，ReplicationController会创建一个新的Pod来替代该Pod，并在正常节点上运行

2. **ReplicationController会持续地监控正在运行的Pod列表，该列表是由ReplicationController中的标签选择器过滤出来的**

   <img src="D:\Program Files\电子书\go\md\图片\image-20230220001019137.png" style="zoom:67%;" />

3. **ReplicationController的三个主要部分**

   - label selector (标签选择器)：用于确定 ReplicationController作用域中有哪些pod
   - replica count(副本个数)：指定应运行的 pod 数量
   - pod template(pod 模板)：用于创建新的 pod 副本

4. 编写一个ReplicationController的Yaml文件

   ```yaml
   apiVersion: v1
   kind: ReplicationController
   metadata:
   	name: kubia
   spec:
   	# 运行的Pod总数
   	replicas: 3
   	# 标签选择器 指定监视app=kubia的Pod
   	# 也可以不指定这个，Kubernetes会自动从template中获取相应的Pod元数据标签进行匹配
   	selector:
   		app: kubia
       template:
       # 模板中的标签必须与ReplicationController的selector匹配
       	metadata:
       		labels:
       			app: kubia
            spec:
   			containers:
   			- name: kubia
   			image: luksa/kubia
   			ports:
   			- containerPort: 8080
   ```

### ReplicaSet

### DaemonSet

### Job——运行执行单个任务的Pod

### CronJob——安排Job定期运行

## 服务：让客户端发现Pod并与之通信

### Service 

#### 介绍Service

- 由于Pod是可以动态地被创建与销毁，那么Kubernetes运行过程中它们的IP可能会变化，因此我们可以定义一个**Service(服务)**，它也是Kubernetes中的一种对象，在创建时被给予了固定的**静态虚拟**IP地址和端口，其次在Service的manifest中，我们可以通过标签选择器`selector`来使得需要通信的Pod与Service进行一种逻辑上的绑定，这样当Pod被删除或重新创建时，携带了`selector`中的标签的Pod就会被Service绑定，当客户端需要与绑定了Service的Pod通信时，可以通过访问Service的IP:端口来进行与Pod的通信。

- 通过创建服务，可以通过单一稳定的IP地址直接访问Pod，在这个服务的整个生命周期内保持不变，

#### 服务发现：Kubernetes给客户端提供发现服务的方式

- 通过环境变量
  - 如果创建的服务早于客户端需要的Pod的创建，那么在Pod中Kubernetes会初始化一系列的环境变量指向现在存在的服务，Pod中的容器应用进程可以通过寻找环境变量来得到Service的IP地址与端口号，若Pod早于某个Service创建，那么Pod中不会有这个Service的环境变量
    - 当 Pod 运行在 `Node` 上，kubelet 会为每个活跃的 Service 添加一组环境变量。 kubelet 为 Pod 添加环境变量 `{SVCNAME}_SERVICE_HOST` 和 `{SVCNAME}_SERVICE_PORT`。 这里 Service 的名称需大写，横线被转换成下划线。
    - **必须在客户端Pod出现之前创建服务，否则Pod中不会有这个Service的环境变量**
- 通过DNS服务(CoreDNS)
  - 在Kubernetes的命名空间`kube-system`中，有这样一个服务——**`kube-dns`**，同时使用deployment创建了coredns容器副本，这个 pod 运行 DNS 服务，在集群中的其他 pod 都被配置成使用其作为 dns(Kubernetes 通过修改每个容器的 /etc/resolv.conf 文件实现)。运行在pod 上的进程DNS 查询都会被 Kubernetes 自身的 DNS 服务器响应，该服务器知道系统中运行的所有服务。
  - 每个客户端的Pod在知道服务名称的情况下可以通过**全限定域名（FQDN）**来访问，而不是通过环境变量
    - 全限定域名指的是`$(服务名).$(命名空间).svc.cluster.local`，

![](D:\Program Files\电子书\go\md\图片\image-20230221142834877.png)

#### 使用Yaml创建服务

```yaml
# svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-test

spec:
  ports:
  # 服务开放的端口
  - name: http
    port: 80
    # 目标端口：即Pod的Yaml文件中暴露的端口，
    targetPort: http
  - name: https
    port: 443
    targetPort: https
  # 标签选择器（选择算符）：用于构建服务的服务列表，然后存储到EndPoint资源中
  selector:
    app: kubia

--- 

apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual
  labels:
    app: zzzyb

spec:
  containers:
  - image: luksa/kubia
    name: kubia
    ports:
    # 为端口号设置名称，解耦
    - name: http
      containerPort: 8080
    - name: https
      containerPort: 8443
```

#### 服务类型

Kubernetes `ServiceTypes` 允许指定你所需要的 Service 类型。`Type` 的取值以及行为如下：

- `ClusterIP`：通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。 这也是你没有为服务显式指定 `type` 时使用的默认值。 你可以使用 [Ingress](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/) 或者 [Gateway API](https://gateway-api.sigs.k8s.io/) 向公众暴露服务。
- [`NodePort`](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#type-nodeport)：通过每个节点上的 IP 和静态端口（`NodePort`）暴露服务。 为了让节点端口可用，Kubernetes 设置了集群 IP 地址，这等同于你请求 `type: ClusterIP` 的服务。
- [`LoadBalancer`](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#loadbalancer)：使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 `NodePort` 服务和 `ClusterIP` 服务上。
- [`ExternalName`](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#externalname)：通过返回 `CNAME` 记录和对应值，可以将服务映射到 `externalName` 字段的内容（例如，`foo.bar.example.com`）。 无需创建任何类型代理。

​	

### EndPointSlice

> Service->EndPoint->Pod

- 在目的上，可能存在需要通过Kubernetes服务特性暴露外部服务的情况，即不要让服务重定向到集群中的Pod，而是让服务重定向到外部IP:端口

- Service并不是与Pod直接相连，有一种资源对象介于两者中间，即`EndPoint资源`，它暴露了Service为之服务的一系列`IP:端口`的组合，同时我们可以创建带有选择算符以及不带有选择算符的Service（不带选择算符的Service就不会绑定某些Pod，也不会创建EndPoint对象），因此我们需要手动配置创建EndPoint

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
  	name: external-service
  spec:
  	ports:
  	- port: 80
  	
  --- 
  
  apiversion: v1
  kind: Endpoints
  metadata:
  # 这里的name需要与Service一致
  name: external-service
  subsets :
  	addresses :
  	# 外部IP，即服务重定向的地址
  	- ip: 11.11.11.11
  	- ip: 22.22.22.22
  	ports:
  	- port: 80
  ```

  <img src="D:\Program Files\电子书\go\md\图片\image-20230221144842415.png" style="zoom:67%;" />

#### 将服务暴露给集群外部客户端

1. 将服务类型设为NodePort

   - 通过创建 NodePort 类型的服务，可以让 Kubernetes 在其所有节点上保留一个端口( 所有节点上都使用相同的端口号 )，并将传入的连接转发给作为服务部分的Pod

     ```yaml
     apiversion: v1
     kind: Service
     metadata :
     name: kubia-nodeport
     spec :
     	type: NodePort
     	ports:
     	# 访问工作节点的端口30123定位到服务的80端口，然后定位到Pod的8080端口
     	- port: 80
     	  targetPort: 8080
     	  nodePort: 30123
     	selector:
     		app: kubia
     ```

     <img src="D:\Program Files\电子书\go\md\图片\1676963495964.png" style="zoom:67%;" />

2. 将服务类型设为LoadBalance

3. 创建一个Ingress资源

### Ingress

#### 介绍Ingress



#### Ingress处理TLS连接



<img src="D:\Program Files\电子书\go\md\图片\1677041920556.png" style="zoom:67%;" />

#### 就绪探针

1. Kubernetes允许为容器定义准备就绪探针，**就绪探针会定期调用，并确定特定的Pod是否接收客户端请求，当容器的准备就绪探针返回成功时，表示容器已准备好接收请求**，对于准备就绪的概念来说，Kubernetes只能检查容器中运行的应用进程是否响应一个简单的GET请求，或者它可以响应特定的URL路径
2. 就绪探针的三种类型
   - Exec探针：执行进程的地方。容器的状态由进程的退出状态代码确定
   - HTTP GET探针：向容器发送HTTP GET 请求，通过响应的HTTP 状态代码0判断容器是否准备好。
   - TCP socket探针：它打开一个TCP 连接到容器的指定端口，如果连接已建立，则认为容器已准备就绪。

3. 向Pod添加就绪探针

   ```yaml
   apiVersion: v1
   kind: ReplicationController
   ...
   spec:
   	...
   	template:
   		...	
   		sepc:
   			containers:
   			- name: kubia
   			  image: luksa/kubia
   			  # 每个容器都有的就绪探针
   			  readinessProbe:
   			  	exec:
   			  		command:
   			  		- ls
   			  		- /var/ready
                   ...
   	
   ```

   - 这样就绪探针将定期在容器内执行`ls /var/ready`命令，若文件存在，则返回退出码0，否则返回非0退出码，即失败

4.  **务必要先定义就绪探针**

   - 首先，如果没有将就绪探针添加到 pod 中，它们几乎会立即成为服务端点。如果应用程序需要很长时间才能开始监听传入连接，则在服务启动但尚未准备好接收传入连接时，客户端请求将被转发到该 pod。因此，客户端会看到“连接被拒绝”类型的错误。
   - **不要把删除Pod的逻辑纳入就绪探针中**，当容器关闭时，并不需要让就绪探针返回失败，才会从服务中删除容器，只需要删除容器，Kubernetes就会从所有服务中移除该容器

### 排除服务故障

如果无法通过服务访问 pod，应该根据下面的列表进行排查 :

- 首先，确保从集群内连接到服务的集群 IP，而不是从外部。
- 不要通过 ping 服务 IP 来判断服务是否可访问 (请记住，服务的集群 IP 是虚拟 IP，是无法 ping 通的)。
- 如果已经定义了就绪探针，请确保它返回成功；否则该 pod 不会成为服务的一部分。
- 要确认某个容器是服务的一部分，请使用 kubectl get endpoints 来检查相应的端点对象。
- 如果尝试通过 FQDN 或其中一部分来访问服务(例如，myservice.mynamespace.svc.cluster.local或myservice.mynamespace)，但并不起作用，请查看是否可以使用其集群 IP 而不是 FODN 来访问服务。
- 检查是否连接到服务公开的端口，而不是目标端口。
- 尝试直接连接到 pod IP 以确认 pod 正在接收正确端口上的连接。
- 如果甚至无法通过 pod 的 IP 访问应用，请确保应用不是仅绑定到本地主机。

 

## 卷：容器如何访问外部磁盘存储

### Volume

- 在Pod的资源文件中，创建数据卷供容器挂载的话，容器中的文件在磁盘上是临时存放的，当容器崩溃时文件会丢失。而kubelet会重启容器，重启之后之前的数据已经不存在，对于有状态的应用来说是致命的。
- Docker中卷的概念是磁盘上或者另一个容器内的一个目录，而Kubernetes支持很多类型的卷。单个容器可以同时使用不同类型的多个卷
- 容器中的进程看到的文件系统视图是由它们的[容器镜像](https://kubernetes.io/zh-cn/docs/reference/glossary/?all=true#term-image) 的初始内容以及挂载在容器中的卷（如果定义了的话）所组成的。 其中根文件系统同容器镜像的内容相吻合。 任何在该文件系统下的写入操作，如果被允许的话，都会影响接下来容器中进程访问文件系统时所看到的内容。

#### 卷的类型

- emptyDir
  - 当 Pod 分派到某个节点上时，`emptyDir` 卷会被创建，并且在 Pod 在该节点上运行期间，卷一直存在。 就像其名称表示的那样，卷最初是空的。 尽管 Pod 中的容器挂载 `emptyDir` 卷的路径可能相同也可能不同，这些容器都可以读写 `emptyDir` 卷中相同的文件。 当 Pod 因为某些原因被从节点上删除时，`emptyDir` 卷中的数据也会被永久删除。
  - 容器崩溃并不会导致 Pod 被从节点上移除，因此容器崩溃期间 `emptyDir` 卷中的数据是安全的。
- hostPath
  - `hostPath` 卷能将主机节点文件系统上的文件或目录挂载到你的 Pod 中。 虽然这不是大多数 Pod 需要的，但是它为一些应用程序提供了强大的逃生舱。
  - **HostPath 卷存在许多安全风险，最佳做法是尽可能避免使用 HostPath。（因为当Pod因某些原因崩溃被重新创建后移动到其他节点上运行，就会导致挂载在之前节点的文件系统上记录的数据丢失）** 当必须使用 HostPath 卷时，它的范围应仅限于所需的文件或目录，并以只读方式挂载。
  - 如果通过 AdmissionPolicy 限制 HostPath 对特定目录的访问，则必须要求 `volumeMounts` 使用 `readOnly` 挂载以使策略生效。
- downwardAPI、configMap、Secret
- persistentVolumeClaim

### PersistentVolume：数据持久化存储

#### PersistentVolume：持久卷

- 集群中的一块存储资源，它同Node一样是集群资源，PV 持久卷和普通的 Volume 一样， 也是使用卷插件来实现的，只是它们拥有独立于任何使用 PV 的 Pod 的生命周期，即当Pod 
- 静态存储的持久卷需要集群管理员(运维人员)自动创建，而动态存储的持久卷不需要手动创建，其可以自适应创建(通过provisioner)

#### PersistentVolumeCliam

- 表达的是用户对存储的请求，是用户对持久卷存储的一个**期望状态**， Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源。Pod 可以请求特定数量的资源（CPU 和内存）；同样 PVC 申领也可以请求特定的大小和访问模式 

- 只需要在`PersistentVolumeCliam`对象中指定需要的资源大小，即在`PersistentVolumeClaim`的`spec.resources.request.storage`指定大小；并将PVC挂载成Pod的一个volume，即在Pod的yaml文件的`spec.volumes.persistentClaim`指定PVC的名称

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: nfs-static-pvc
  
  spec:
    # 通过对象名字指定storageClass
    storageClassName: nfs
    accessModes:
    	# 卷可以被多个节点以读写方式挂载
      - ReadWriteMany
  
    resources:
      requests:
        storage: 1Gi
  
  ---
  
  apiVersion: v1
  kind: Pod
  metadata:
    name: ...
  
  spec:
    volumes:
    - name: nfs-pvc-vol
      persistentVolumeClaim:
        claimName: nfs-static-pvc
  
    containers:
      - name: ...
        image: ...
        ports:
        - containerPort: 80
  
        volumeMounts:
        # volume's name
          - name: nfs-pvc-vol
            mountPath: ...
  ```

  

#### 访问模式

> PersistentVolume 卷可以用资源提供者所支持的任何方式挂载到宿主系统上。 如下表所示，提供者（驱动）的能力不同，每个 PV 卷的访问模式都会设置为对应卷所支持的模式值。

- `ReadWriteOnce`：卷可以被一个节点以读写方式挂载。 ReadWriteOnce 访问模式也允许运行在同一节点上的多个 Pod 访问卷。

- `ReadOnlyMany`：卷可以被多个节点以只读方式挂载。

- `ReadWriteMany`：卷可以被多个节点以读写方式挂载。

- `ReadWriteOncePod`：卷可以被单个 Pod 以读写方式挂载。 如果你想确保整个集群中只有一个 Pod 可以读取或写入该 PVC， 请使用 ReadWriteOncePod 访问模式。这只支持 CSI 卷以及需要 Kubernetes 1.22 以上版本。

#### StorageClass：动态存储

> **让创建PV的工作实现自动化，而不需要集群管理员每次在Pod需要时之前手动进行创建，可以用 StorageClass 绑定一个 Provisioner 对象，而这个 Provisioner 就是一个能够自动管理存储、创建 PV 的应用，代替了原来系统管理员的手工劳动。**

- StorageClass 为管理员提供了描述存储 "类" 的方法。 不同的类型可能会映射到不同的服务质量等级或备份策略，或是由集群管理员制定的任意策略。

- 每个 StorageClass 都包含 `provisioner`、`parameters` 和 `reclaimPolicy` 字段， 这些字段会在 StorageClass 需要动态制备 PersistentVolume 时会使用到。

- **provisioner**

  Kubernetes 里每类存储设备都有相应的 Provisioner 对象，可在Github的k8s（https://github.com/kubernetes-sigs/）上找到对应的存储设备的Provisioner，通过相应的资源文件部署了StorageClass和provisioner后，可以直接在PVC中指定StorageClass对象，它再关联到Provisioner即可使用对应的存储设备

  ![](https://static001.geekbang.org/resource/image/e3/1e/e3905990be6fb8739fb51a4ab9856f1e.jpg?wh=1920x856)

  ```yaml
  
  ```

  

#### 静态存储：手动创建PV	

​	集群管理员创建若干 PV 卷。这些卷对象带有真实存储的细节信息， 并且对集群用户可用（可见）。PV 卷对象存在于 Kubernetes API 中，可供用户消费（使用）。

#### 回收持久卷

当用户不再使用其存储卷时，他们可以从 API 中将 PVC 对象删除， 从而允许该资源被回收再利用。PersistentVolume 对象的回收策略告诉集群， 当其被从申领中释放时如何处理该数据卷。 目前，数据卷可以被 Retained（保留）、或 Deleted（删除）。

- **手动回收持久卷**：通过将persistentVolumeReclaimPolicy设置为Retain 从而通知到Kubernetes，我们希望在创建持久卷后将其持久化，让 Kubernetes 可以在持久卷从持久卷声明中释放后仍然能保留它的卷和数据内容。据我所知，手动回收持久卷并使其恢复可用的唯一方法是删除和重新创建持久卷资源。当这样操作时，你将决定如何处理底层存储中的文件:可以删除这些文件，也可以闲置不用，以便在下一个pod 中复用它们。
- **自动回收持久卷**：对于支持 `Delete` 回收策略的卷插件，删除动作会将 PersistentVolume 对象从 Kubernetes 中移除，同时也会从外部基础设施（如 AWS EBS、GCE PD、Azure Disk 或 Cinder 卷）中移除所关联的存储资产。 动态制备的卷会继承[其 StorageClass 中设置的回收策略](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/#reclaim-policy)， 该策略默认为 `Delete`。管理员需要根据用户的期望来配置 StorageClass； 否则 PV 卷被创建之后必须要被编辑或者修补。 





