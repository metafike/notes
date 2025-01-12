# 容器持久化存储

![](image/2022-08-22-10-51-53.png)

容器的本质是进程，对于进程，`Linux`系统有进程组的概念来将其组织在一起。在`k8s`里面，使用`Pod`这个逻辑概念来维护容器间的关系。

有了`Pod`后，我们的应用程序需要被创建和管理，这就引出了`ReplicaSet`和`Deployment`；然后需要将部署好的应用暴露给外部进行访问，`Service`可以提供一个固定的ip和端口让外部访问。

对于有状态的应用，可以使用`StatefulSet`来进行状态的恢复，在上一节[概念介绍](https://www.howie6879.cn/p/k8s%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF.02.%E6%A6%82%E5%BF%B5%E4%BB%8B%E7%BB%8D/)里面有提到，有状态的应用是离不开**持久化存储**的。

## 引子

在`Docker`中，如果一个容器在运行过程中会产生数据并写入到文件系统，当关闭这个容器，用镜像再启动一个容器的时候，你就会意识到新容器并不会识别前一个容器写入文件系统内的任何内容。

对于有状态的应用，我们希望下次启动的应用可以保持住上次的状态；在`k8s`里面可以通过定义**存储卷**来满足这个需求，它们不像`Pod`这样的顶级资源，而是被定义为`Pod`的一部分，并和`Pod`共享相同的生命周期。因此在`Pod`里面容器重新启动期间，卷的内容是不变的，

## 卷

### emptyDir

在`Pod`中如何定义卷？让我们从`emptyDir`开始。设想一个这样的例子，一个`Pod`应用由两个容器，容器A不断产生数据，容器B将A产生的数据作为输出。此时，这两个容器就需要使用同一个卷。

让我们实际操作一下，`vim fortune-pod.yaml`:

```yaml

apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {}
```

说明一下上述配置文件的含义：`fortune`镜像是`k8s in action`书中示例打包的镜像，相当于上面说的不断产生数据的容器A，其中名为`html`的容器挂载在`var/htdocs`中；而`nginx`也挂载了相同的`html`卷，不过位置在`/usr/share/nginx/html`，上面两个容器共用的卷就是`emptyDir: {}`。

启动来感受一下：

```shell
kubectl create -f fortune-pod.yaml
# 输出
pod/fortune created

# 查看状态
kubectl get pods
# 输出
NAME      READY   STATUS    RESTARTS   AGE
fortune   2/2     Running   0          2m27s

# 暂时服务化
kubectl port-forward fortune 8080:80
```

此时服务就处于可用状态了，在终端输入`curl http://localhost:8080/`，基本上每隔`10s`，都会返回不同的响应，如下图：

![image-20210305144115418](https://images-1252557999.file.myqcloud.com/uPic/image-20210305144115418.png)

`emptyDir`卷是最简单的卷类型，但是其他类型的卷都是在它的基础上构建的，在创建空目录后，相应的容器会将数据写入。

### gitRepo

假设你有在`github`上开发项目，`gitRepo`卷允许你定义好相关配置然后直接从`github`上下拉项目将数据共享给其他容器使用。

### hostPath

前面说的卷都是停留在共享同一个`Pod`的文件，当其需要读取节点文件的时候，就需要`hostPath`卷出场了。和之前介绍的卷最大的不同之处是，`hostPath`是一个持久性存储的卷，其目录存在于对应节点主机的目录。

所以，`hostPath`仅仅适用于在节点上读取数据，如果你的需求是跨`Pod`，那么`NAS`才是你的解决方案。

### NFS

目前相关的云商都会有一套自己的持久化方式，我目前是自己搭建的`k8s`，所以我只能实践一下`NAS`方案，新建文件`vim mongodb-pod-nfs.yaml`，输入以下内容：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb-nfs
spec:
  volumes:
  - name: mongodb-data
    nfs:
      server: 1.2.3.4
      path: /some/path
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
```

启动：

```shell
kubectl create -f mongodb-pod-nfs.yaml

# 查看状态
kubectl get pods
# 输出
NAME          READY   STATUS    RESTARTS   AGE
mongodb-nfs   1/1     Running   0          3m13s
```

我们来验证一下数据持久化是否生效，输入命令`kubectl exec -it mongodb-nfs mongo`进入：

```shell
> use test_data
switched to db test_data

# 插入
> db.test.insert({"name": "howie"})
WriteResult({ "nInserted" : 1 })

# 查询
> db.test.find({})
{ "_id" : ObjectId("6041f5bc0c893dc3bb362e75"), "name" : "howie" }
```

接下来重新创建`Pod`看一下数据是不是还在：

```shell
kubectl delete pod mongodb-nfs

# 重新创建
kubectl create -f mongodb-pod-nfs.yaml
# 输出
pod/mongodb-nfs created

# 查看
kubectl get pods
# 输出
NAME          READY   STATUS    RESTARTS   AGE
mongodb-nfs   1/1     Running   0          28s
```

进入`Pod`内的`mongo`容器，输入命令`kubectl exec -it mongodb-nfs mongo`进入：

```shell
> use test_data
switched to db test_data

# 查询
> db.test.find({})
{ "_id" : ObjectId("6041f5bc0c893dc3bb362e75"), "name" : "howie" }
```

虽然我们成功让多个`Pod`享用了同一份数据，但这样做法有点问题，让开发人员在配置里面写具体`NFS`地址是很不友好的事情，我们可以使用**持久卷**来解决此问题。

## 持久卷&持久卷声明

### 介绍

前面`NFS`用来做持久化存储是一个反面的例子，对于真实的基础设施，其详细配置应该是被隐藏的；但是`k8s`又实实在在需要对一些基础设施进行访问，怎么办？引入新的资源：

- 持久卷（PersistentVolume）
- 持久卷声明（PersistentVolumeClaim）

持久卷由管理员创建（各种配置信息），然后用户创建持久卷声明，提交后`k8s`就会找到匹配的持久卷并将其绑定到持久卷声明。

![](image/2023-08-09-16-49-09.png)

这样做的好处在于，对于用户只需要关注声明一下需要多大的存储、需要什么权限（读写）等，然后pod通过其中一个卷的名称来引用声明就可以了，将细节完美地进行了隐藏。

### 实践

首先建立一个`NFS`类型的`PV`（一般是管理员进行创建），在终端输入`vim mongodb-pv-nfs.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 10Mi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 1.2.3.4
    path: "/"
```

接下来创建持久卷：

```shell
kubectl create -f mongodb-pv-nfs.yaml
# 输出
persistentvolume/nfs created

# 查看
kubectl get pv
# 输出
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
nfs-pv   10Mi       RWX            Retain           Available                                   6s

# 关于删除
# kubectl delete pv nfs-pv
```

可以看到状态已经生效。

接下来就轮到使用者随意使用`PV`了，如果作为使用者，部署的`Pod`需要持久化存储，那么其需要做的就是创建`PVC`，在终端输入`vim mongodb-pvc-nfs.yaml`：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
  storageClassName: ""
```

然后创建持久化声明：

```shell
kubectl create -f mongodb-pvc-nfs.yaml
# 输出
persistentvolumeclaim/nfs created

# 查看 pvc
kubectl get pvc
# 输出，注意状态是绑定
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc   Bound    nfs-pv   10Mi       RWX                           6s

# 查看 pv
kubectl get pv
# 输出，状态是绑定
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
nfs-pv   10Mi       RWX            Retain           Bound    default/nfs-pvc                           3m6s
# 关于删除
# kubectl delete pvc nfs-pvc
```

对于`ACCESS MODES`，主要分为以下几种：

- RWO：ReadWriteOnce（仅允许单个节点挂载读写）
- ROX：ReadOnlyMany（允许多个节点挂载只读）
- RWX：ReadWriteMany（允许多个节点挂载读写这个卷）

现在，准备工作就绪，在`Pod`中使用持久卷就是引用持久卷名称，在终端输入`vim mongo-pod-pvc.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: mongodb-data
    persistentVolumeClaim:
      claimName: nfs-pvc
```

创建`Pod`：

```shell
# 先删除原先创建的 mongo Pod
kubectl delete pod mongodb-nfs
# 创建引用持久卷声明的 Pod
kubectl create -f mongodb-pod-pvc.yaml
# 输出
pod/mongodb created
```

进入`Pod`内的`mongo`容器，输入命令`kubectl exec -it mongodb mongo`进入：

```shell
> use test_data
switched to db test_data

# 查询
> db.test.find({})
{ "_id" : ObjectId("6041f5bc0c893dc3bb362e75"), "name" : "howie" }
```

没问题，引用了之前的`NFS`下的对应目录。

## 动态卷

前面提到，`PV`需要管理人员进行创建，在实际生产环境下，这个`PV`的需求量可能是非常大的，所以这种协调方式是不合理的。所以，`k8s`提供了一套可以自动创建`PV`的机制——**动态卷**。

![image-20210305213129142](https://images-1252557999.file.myqcloud.com/uPic/image-20210305213129142.png)

这张图将使用`StorageClass`的流程描述地很清楚，管理员创建一个或多个`StorageClass`，用户创建`Pod`引用`PVC`声明相关的`storageClassName`就会通过管理员创建的`StorageClass`自动创建`PV`。

## 参考

本章关于容器持久化就介绍到这里了，谢谢！本部分内容有参考如下文章：

- [深入剖析Kubernetes](https://time.geekbang.org/column/intro/100015201?code=UhApqgxa4VLIA591OKMTemuH1%2FWyLNNiHZ2CRYYdZzY%3D)：持久化存储部分
- [Kubernetes in Action](https://github.com/luksa/kubernetes-in-action)中文版：可以算是第6章的读书笔记

