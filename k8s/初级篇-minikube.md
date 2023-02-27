# minikube

## 安装

```shell
curl -LO https://github.com/kubernetes/minikube/releases/download/v1.25.2/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube


```





## 使用minikube

1. 启动

```shell
# 部分地区若没使用--registry-mirror会导致拉取镜像时容器一直在创建状态
minikube start --kubernetes-version=v1.23.3 --image-mirror-country='cn' --registry-mirror=https://9q1nmamk.mirror.aliyuncs.com
```

Kubernetes不支持Pod当前状态的停止/暂停以及在需要时恢复。

## 配置文件ConfigMap和Secret

1. 通过编写yaml文件，可以创建ConfigMap和Secret对象

   ```shell
   
   # 创建
   kubectl apply -f *.yaml
   
   # 查看
   kubectl get cm name
   kubectl get secret name
   ```

2. Secret的key-value存储时，value按照base64进行编码

3. 如何将配置文件应用到Pod的的Container中

   1. 使用环境变量

      1. 描述容器字段`containers`有一个`env`，定义了Pod里容器能够看到的环境变量，

         ```yaml
         
         apiVersion: v1
         kind: Pod
         metadata:
           name: env-pod
         
         spec:
           containers:
           - env:
               - name: COUNT
         # 容器的env中name为COUNT的值来自name为info的configMap中的key为count的字段
                 valueFrom:
                 # configMapKeyRef
                   configMapKeyRef:
                     name: info
                     key: count
               - name: PASSWORD
                 valueFrom:
                   secretKeyRef:
                     name: user
                     key: pwd
         
             image: busybox
             name: busy
             imagePullPolicy: IfNotPresent
             command: ["/bin/sleep", "300"]
         ```

         

   2. 通过Volume的方式使用ConfigMap/Secret

      1. 先在Pod的yaml文件中定义两个Volume
      2. 定义完，在Pod的yaml文件中使用**`volumeMounts`**，可以把定义好的Volume挂载到容器里的某个路径下，需要在其中用`mountPath`\ `mame`明确地确定挂载路径和Volume的名字
      3. 
