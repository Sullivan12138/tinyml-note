## k8s创建容器流程
我们创建好镜像以后，在之前 Docker 环境下面我们是直接通过命令 docker run 来运行我们的应用的，在 Kubernetes 环境下面我们同样也可以用类似 kubectl run 这样的命令来运行我们的应用，但是在 Kubernetes 中却是不推荐使用命令行的方式，而是希望使用我们称为资源清单的东西来描述应用，资源清单可以用 YAML 或者 JSON 文件来编写，一般来说 YAML 文件更方便阅读和理解。

通过一个资源清单文件来定义好一个应用后，我们就可以通过 kubectl 工具来直接运行它：
```
$ kubectl create -f xxxx.yaml
```
我们知道 kubectl 是直接操作 APIServer 的，所以就相当于把我们的清单提交给了 APIServer，然后集群获取到清单描述的应用信息后存入到 etcd 数据库中，然后 kube-scheduler 组件发现这个时候有一个 Pod 还没有绑定到节点上，就会对这个 Pod 进行一系列的调度，把它调度到一个最合适的节点上，然后把这个节点和 Pod 绑定到一起（写回到 etcd），然后节点上的 kubelet 组件这个时候 watch 到有一个 Pod 被分配过来了，就去把这个 Pod 的信息拉取下来，然后根据描述通过容器运行时把容器创建出来，最后当然同样把 Pod 状态再写回到 etcd 中去，这样就完成了一整个的创建流程。

## job
容器按照持续运行的时间可分为两类：服务类容器和工作类容器。
服务类容器通常持续提供服务，需要一直运行，比如HTTP Server、Daemon等。工作类容器则是一次性任务，比如批处理程序， 完成后容器就退出。
Kubernetes的Deployment、ReplicaSet和DaemonSet都用于管理服务类容器；对于工作类容器，我们使用Job。

Kubernetes支持以下几种Job：
- 非并行Job：通常创建一个Pod直至其成功结束
- 固定结束次数的Job：设置.spec.completions，创建多个Pod，直到.spec.completions个Pod成功结束
- 带有工作队列的并行Job：设置.spec.Parallelism但不设置.spec.completions，当所有Pod结束并且至少一个成功时，Job就认为是成功

特点

RestartPolicy仅支持Never或OnFailure
单个Pod时，默认Pod成功运行后Job即结束
.spec.completions标志Job结束需要成功运行的Pod个数，默认为1
.spec.parallelism标志并行运行的Pod的个数，默认为1
spec.activeDeadlineSeconds标志失败Pod的重试最大时间，超过这个时间不会继续重试

一个job的yaml文件示例如下：
```
[root@k8s-master ~]# cat myjob.yml 
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  template:
    metadata:
      name: myjob
    spec:
      containers:
      - name: hello
        image: busybox
        command: ["echo", "hello k8s job! "]
      restartPolicy: Never
[root@k8s-master ~]# 
```
其中，job的apiVersion一般为batch/v1，restartPolicy指定什么情况下需要重启容器。对于Job，只能设置为Never或者OnFailure。对于其他controller（比如Deployment），可以设置为Always。
通过kubectl apply -f myjob.yml启动Job，通过kubectl get job查看Job的状态，通过kubectl logs可以查看Pod的标准输出。

