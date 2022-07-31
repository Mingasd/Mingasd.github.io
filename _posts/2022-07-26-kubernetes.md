---
layout: post
title: Kubernates
data: 2022-07-26
tagss: [分布式]
---

### Kubernetes是什么

它是一个为 容器化 应用提供集群部署和管理的开源工具，由 Google 开发。



### Kubernetes的集群架构

![](https://raw.githubusercontent.com/Mingasd/PostImg/main/kebernetes%E9%9B%86%E7%BE%A4%E6%9E%B6%E6%9E%84.png)

主要分为两部分，主节点和工作节点。

- 主节点：控制平台，不需要很高性能，不跑任务，通常一个就行了，也可以开多个主节点来提高集群可用度。

- 工作节点：可以是虚拟机或物理计算机，任务都在这里跑，机器性能需要好点；通常都有很多个，可以不断加机器扩大集群；每个工作节点由主节点管理。

  

Kubernetes中调度的最小单位是Pod，一个Pod中可以包含一个或多个容器，Pod 里的容器共享同一个 Network Namespace、同一组数据卷，从而达到高效率交换信息的目的，每个Pod都有自己的虚拟ip，一个工作节点Node可以存放多个Pod。



### kubernetes里面的组件

- Kube-apiserver：API服务器，公开Kubernetes API
- Etcd：键值数据库
- Kube-scheduler：调度Pod到哪个节点运行
- kube-controller： 集群控制器，负责容器的编排
- cloud-controller： 与云服务商交互



Kubernetes有多个对象，之前所说的**pod**也是一种对象，除此之外还有**deploymenet**、**statefulset**、**service**、**configmap**、**secret**、**ingress**等等。这些对象可以统一的使用ymal文件进行配置，可以说是面向yaml编程了。

#### Pod

现在我们先看一下pod对象该怎么描述

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  # 定义容器，可以多个
  containers:
    - name: test-k8s # 容器名字
      image: ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v1 # 镜像

```

大致就是这样描述，kind字段标注这个对象是pod，spec字段里说明了这个pod对象里面包含一个容器，容器名称在name字段，image字段标注了容器镜像地址。

但是，假设现在我们有很多的服务，每个服务都需要部署多次，那么一个个地对pod进行部署就会很麻烦，对此引入了deployment和statefulset这两个对象。

#### Deployment

Deployment适用于无状态的服务，也就是那些不需要持久化数据的服务，不需要记住某一时刻的状态，可以随意扩充副本，每个副本都可以被替代。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  # 部署名字
  name: test-k8s
spec:
  replicas: 2
  # 用来查找关联的 Pod，所有标签都匹配才行
  selector:
    matchLabels:
      app: test-k8s
  # 定义 Pod 相关数据
  template:
    metadata:
      labels:
        app: test-k8s
    spec:
      # 定义容器，可以多个
      containers:
      - name: test-k8s # 容器名字
        image: ccr.ccs.tencentyun.com/k8s-tutorial/test-k8s:v1 # 镜像

```

通过这个yaml文件可以对deployment对象进行一个描述，kind标注这个yaml是一个deployment对象，紧接着spec里面对这个deployment进行了描述，replicas字段表面这个服务有2个副本，selector中标注这个ReplicaSet对象将会匹配哪个pod，这个pod正式在下面那个template中定义的，里面的内容和上面那个pod一样。

Deployment 的控制器，实际上控制的是 ReplicaSet 的对象，具体的pod是由replicaSet控制的。

![](https://raw.githubusercontent.com/Mingasd/PostImg/main/deployment%E7%BB%93%E6%9E%84.png)

Deployment对象采用水平扩展/收缩和滚动更新两个编排动作。

- 水平扩展和收缩：增加或减少集群中对应的pod数量。
- 滚动更新：当我们容器的镜像更新时，新pod副本一个个水平扩展，旧pod副本一个个水平收缩。

#### StatefulSet

StatefulSet 是用来管理有状态的应用，例如数据库。StatefulSet 会固定每个 Pod 的名字，在重启后，StatefulSet所包括的Pod名字是不变的，但Pod的IP是不固定的。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongo
          image: mongo:4.4
          # IfNotPresent 仅本地没有镜像时才远程拉，Always 永远都是从远程拉，Never 永远只用本地镜像，本地没有则报错
          imagePullPolicy: IfNotPresent
```

#### Service

Pod虽然有IP，但确实不固定的，某个Pod被重启后将会分配不同的IP，则就使得如何访问Pod变成一个难题。由此出现service服务，service通过lable与Pod关联，service的访问是固定的，访问service后，service会通过负载均衡选个一个Pod，将请求转发给这个Pod。

- Service 通过 label 关联对应的 Pod
- Servcie 生命周期不跟 Pod 绑定，不会因为 Pod 重创改变 IP
- 提供了负载均衡功能，自动转发流量到不同 Pod
- 可对集群外部提供访问端口
- 集群内部可通过服务名字访问

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-k8s
spec:
  selector:
    app: test-k8s
# 默认 ClusterIP 集群内可访问，NodePort 节点可访问，LoadBalancer 负载均衡模式（需要负载均衡器才可用）
  type: ClusterIP
  ports:
    - port: 8080        # 本 Service 的端口
      targetPort: 8080  # 容器端口
```

由上面的yaml文件可以看出，这个service代理了test-k8s这些Pod，type字段标明这个service是ClusterIP，也就是它是一个虚拟IP，VIP，只能在集群内访问这个service，集群外部是ping不通的；如果想让外部可以访问到这个service，那么service的Type就需要标明为NodePort。有时不需要或不想要负载均衡，以及单独的 Service IP。 遇到这种情况，可以通过指定 Cluster IP（`spec.clusterIP`）的值为 `"None"` 来创建 `Headless` Service，通常数据库配置为Headless Service。

#### Ingress

可以看作是service的service。功能类似 Nginx，可以根据域名、路径把请求转发到不同的 Service。

![](https://raw.githubusercontent.com/Mingasd/PostImg/main/ingress.png)

要使用 Ingress，需要一个负载均衡器 + Ingress Controller。负载均衡器云服务商，会自动给你配置。

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-example
spec:
  ingressClassName: nginx
  rules:
  - host: enos.iot
    http:
      paths:
      - path: /hello
        pathType: Prefix
        backend:
          service:
            name: service1
            port:
              number: 4200
      - path: /world
        pathType: Prefix
        backend:
          service:
            name: service2
            port:
              number: 8080

```

#### ConfigMap&Secret

数据库连接地址，这种可能根据部署环境变化的，我们不应该写死在代码里。Kubernetes 为我们提供了 ConfigMap，可以方便的配置一些变量。[文档](https://kubernetes.io/zh-cn/docs/concepts/configuration/configmap/)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-config
data:
  mongoHost: mongodb-0.mongodb
```

这样就配置除了一个key-value，mongoHost->mongodb-0.mongodb

一些重要数据，例如密码、TOKEN，我们可以放到 secret 中。[文档](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongo-secret
# Opaque 用户定义的任意数据，更多类型介绍 https://kubernetes.io/zh/docs/concepts/configuration/secret/#secret-types
type: Opaque
data:
  # 数据要 base64
  mongo-username: bW9uZ291c2Vy
  mongo-password: bW9uZ29wYXNz
```

Secret 可以以数据卷的形式挂载，也可以作为[环境变量](https://kubernetes.io/zh-cn/docs/concepts/containers/container-environment/) 暴露给 Pod 中的容器使用。

下面是一个通过卷来挂载名为 `mysecret` 的 Secret 的 Pod 示例：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - name: mongodb
    image: mongo:4.4
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mongo-secret
      optional: false # 默认设置，意味着 "mysecret" 必须已经存在
```

下面是在使用环境变量的方式使得Pod使用Secret

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongo
          image: mongo:4.4
          # IfNotPresent 仅本地没有镜像时才远程拉，Always 永远都是从远程拉，Never 永远只用本地镜像，本地没有则报错
          imagePullPolicy: IfNotPresent
          env:
          - name: MONGO_INITDB_ROOT_USERNAME
            valueFrom:
              secretKeyRef:
                name: mongo-secret
                key: mongo-username
          - name: MONGO_INITDB_ROOT_PASSWORD
            valueFrom:
              secretKeyRef:
                name: mongo-secret
                key: mongo-password
          # Secret 的所有数据定义为容器的环境变量，Secret 中的键名称为 Pod 中的环境变量名称
          # envFrom:
          # - secretRef:
          #     name: mongo-secret

```

#### 数据持久化

kubernetes 集群不会为你处理数据的存储，我们可以为数据库挂载一个磁盘来确保数据的安全。目前使用pvc pv结构来实现数据的持久化。

![](https://raw.githubusercontent.com/Mingasd/PostImg/main/PVC%E6%9E%B6%E6%9E%84.png)

PV(Persistent Volume)描述卷的具体信息，例如磁盘大小，[访问模式](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#access-modes)。[文档](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/)，[类型](https://kubernetes.io/zh/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes)，[Local 示例](https://kubernetes.io/zh/docs/concepts/storage/volumes/#local)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongodata
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem  # Filesystem（文件系统） Block（块）
  accessModes:
    - ReadWriteOnce       # 卷可以被一个节点以读写方式挂载
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /root/data
  nodeAffinity:
    required:
      # 通过 hostname 限定在某个节点创建存储卷
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - node2

```

PVC(Persistent Volume Claim)是一个PV的声明，可以理解为一个申请单，系统根据这个申请单去找一个合适的 PV。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodata
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: "local-storage"
  resources:
    requests:
      storage: 2Gi

```

运维人员负责提供好存储，开发人员不需要关注磁盘细节，只需要写一个PVC申请单。