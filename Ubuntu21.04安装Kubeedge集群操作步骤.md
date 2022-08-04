# Ubuntu21.04安装Kubeedge集群操作步骤

## 版本要求

系统最好选择能支持Glibc3.2以上的，我之前在Ubuntu16.04上安装就失败了，原因是系统支持的Glibc版本不够，而升级Glibc又是一件非常麻烦且一不小心就会导致系统崩溃的事，所以不如直接选择一个高版本的系统。Kubernetes我选择的是v1.18.0，不能选择太新的版本，一开始我选择的是v1.22.2，后面发现这个版本对应的docker image太新了以至于在国内的镜像站找不到。Kubeedge我选择的是v1.6.0，需要注意**Kubeedgev1.3.0之前和之后的安装方法是不一样的！！！**网上很多教程都没有提到这一点，所以你会看到网上的教程安装方法都不一样，那是因为他们对应的版本不同。

## 事先准备

需要准备一台电脑作为master（云端），另一台作为node（边缘端）节点，当然一个生产环境的集群至少要3台机器，不过我们只是要示范性的搭个集群，所以两台就够了。两台电脑要在一个子网内。

## 具体步骤

### 主节点开启Kubernetes集群

云端和边缘端都需要安装docker，云端需要安装Kubernetes，边缘端不需要。接下来的操作我会在注释中表明这是在云端还是边缘端还是在二者上都要进行的操作。

安装的时候，虽然很多命令都要用root权限，但建议在自己的用户身份下操作，**不要为了省事用`sudo su`切换到root用户去操作**！因为root用户和你自己的用户本质上是两个用户，很多东西都不一样。举个例子，你自己用户的环境变量和root用户的环境变量是不一样的，你切换到root以后，很多需要环境变量的操作很可能就会出错。

首先修改Ubuntu的软件源，添加国内的镜像源，这样才能顺利把docker下载下来。

```bash
# 云端和边缘端
# 备份原来的源
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo vim /etc/apt/sources.list
# 将以下内容写到该文件末尾
deb http://mirrors.aliyun.com/ubuntu/ xenial main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main
 
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main
 
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
 
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security universe
# 然后进行以下操作
sudo apt update
sudo apt -y install docker.io
```

然后，在云端主机上，关闭swap分区功能和防火墙，具体操作是：

```bash
# 云端
sudo sed -i '/ swap / s/^/#/' /etc/fstab
sudo swapoff -a
sudo ufw disable
```

然后安装kubeadm、kubelet、kubectl

```bash
# 云端
# 安装必要的依赖
sudo apt update
sudo apt install -y apt-transport-https
# 添加kubernetes的软件源
sudo vim /etc/apt/sources.list.d/kubernetes.list # 如果没有sources.list.d就先创建这个文件夹
# 将以下内容写到该文件中
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
# 然后执行以下操作
sudo apt update
```

不出意外的话，上一步操作进行完了以后会报错，报错内容是`no PUBKEY XXXXXX`，这时需要执行以下操作

```bash
# 云端
gpg --keyserver keyserver.ubuntu.com --recv <PUBKEY># 此处填报错的公钥
gpg --export --armor <PUBKEY> | sudo apt-key add - # 同上
# 有几个报错的公钥就执行几遍，全部执行完以后再执行以下操作
sudo apt update
sudo apt install -y kubelet=1.18.0-00 kubeadm=1.18.0-00  kubectl=1.18.0-00
```

然后用kubeadm初始化Kubernetes集群

```bash
# 云端
sudo kubeadm init --image-repository=registry.aliyuncs.com/google_containers --kubernetes-version=v1.18.0
```

如果看到以下语句，说明初始化成功

```bash
kubeadm join 192.168.0.49:6443 --token fkxju7.d39l2sct5bc4w5yo \
    --discovery-token-ca-cert-hash sha256:28b467ec8f97537069724028c5d51650983b8bbc2ac29a6e52b210bb2d1896ff 
```

其中的token，如果你需要后面其他节点以Kubernetes node的角色加入这个集群，那么你要记下来。这里我们其他节点是以Kubeedge的角色加入，所以记不记无所谓。

如果这一步初始化失败了，可能是你之前初始化过，再去初始化就会失败。这个时候需要先`sudo kubeadm reset`，然后再初始化即可。如果还是不行，那么就是你之前初始化的到现在的期间你的电脑ip变了，这个时候只能把原来的配置文件都删掉再继续初始化，具体是执行以下操作

```bash
# 云端
sudo kubeadm reset
sudo systemctl stop kubelet
sudo systemctl stop docker
sudo rm -rf /var/lib/cni/
sudo rm -rf /var/lib/kubelet/*
sudo rm -rf /etc/cni/
sudo rm -rf /etc/kubernetes/*
sudo systemctl restart kubelet
sudo systemctl restart docker
```

初始化成功以后，执行以下操作

```bash
# 云端
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

这个时候可以执行`kubectl get pods -A`查看所有pod是否变成ready，正常情况下应该有几个pod还没有ready，这是因为还没有配置网络插件。执行以下操作

```bash
# 云端
kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
```

再等待一段时间，再执行`kubectl get pods -A`，这时所有pod都变成ready了，那么就成功建立起Kubernetes集群了，后面就是其他节点加入这个集群了。可以执行`kubectl get nodes`查看节点情况，这个时候应该只有一个master节点，并且是ready状态。

### 主节点开启Kubeedge cloud服务

这里我们只讲Kubeedge v1.3.0以后的安装方法，因为很多模型运行要求都到了v1.5以后，再安装以前版本的Kubeedge显得没有意义了。

先安装golang，运行以下命令

```bash
# 云端
git clone https://golang.google.cn/dl/go1.15.6.linux-amd64.tar.gz
tar -zxvf go1.15.6.linux-amd64.tar.gz -C /usr/local 
#配置用户环境
vim ~/.bashrc
#文件末尾加上
export PATH=$PATH:/home/ubuntu/tools/go/bin
export GOROOT=/home/ubuntu/kubeedge
export GOPATH=/home/ubuntu/kubeedge
#刷新
source ~/.bashrc
#检查go环境
go version
```

然后安装gcc，make

```bash
# 云端
sudo apt install make
sudo apt install gcc
```

然后需要从github下载kubeedge的源码与编译好的二进制代码

```bash
# 云端
# 下载编译好的二进制代码
wget https://github.com/kubeedge/kubeedge/releases/download/v1.6.0/kubeedge-v1.6.0-linux-amd64.tar.gz
tar -xzvf kubeedge-v1.3.1-linux-amd64.tar.gz
mv kubeedge-v1.3.1-linux-amd64 $GOPATH/src/github.com/kubeedge/kubeedge-v1.3.1
# 下载源码
git clone https://github.com/kubeedge/kubeedge.git $GOPATH/src/github.com/kubeedge/kubeedge
```

如果觉得github太慢了甚至连不上，可以去gitee上面把这个repo导入到gitee里，然后下载gitee上的repo，具体操作可以百度，这里不再赘述了。

Kubeedge1.3.0之后不需要再手动生成证书了，如果是以前的版本则需要，如果你之前安装了以前版本的，这里需要把证书删掉。`ls /etc/kubeedge/` 看看是否有ca和certs两个文件夹，如果有就把它们删掉。如果你之前没有安装以前版本的可以不用做这一步。

然后需要创建设备模块和设备CRD yaml 文件。执行以下操作

```bash
# 云端
cd $GOPATH/src/github.com/kubeedge/kubeedge/build/crds/devices
kubectl create -f devices_v1alpha1_devicemodel.yaml
kubectl create -f devices_v1alpha1_device.yaml

cd $GOPATH/src/github.com/kubeedge/kubeedge/build/crds/reliablesyncs
kubectl create -f cluster_objectsync_v1alpha1.yaml
kubectl create -f objectsync_v1alpha1.yaml
```

然后开始配置云端节点，执行以下操作

```bash
# 云端
cd $GOPATH/src/github.com/kubeedge/kubeedge-v1.3.1/cloud/
sudo su
mkdir -p /etc/kubeedge/config/ 
cloudcore --minconfig > /etc/kubeedge/config/cloudcore.yaml 
exit
```

然后修改生成的cloudcore.yaml文件

```bash
# 云端
sudo vim /etc/kubeedge/config/cloudcore.yaml 
```

修改其中kubeConfig值为/home/你的用户名

然后运行`sudo ./cloudcore`开启cloud服务

然后它会一直运行下去，可以执行以下命令

```bash
# 云端
sudo su
nohup ./cloudcore > cloudcore.log 2>&1 &
```

将其挂起在后台运行，并将输出重定向到cloudcore.log

### 边缘端开启edge服务

可以直接将云端配置好的edgecore文件用scp命令传输到边缘端，需要先在边缘端开启ssh服务。执行以下操作

```bash
# 边缘端
sudo apt install ssh
sudo systemctl restart ssh
```

然后下载mosquitto（只有边缘端需要mosquitto），执行以下操作

```bash
# 边缘端
sudo add-apt-repository ppa:mosquitto-dev/mosquitto-ppa 
sudo apt update 
# 这一步执行完后可能同样会出现前面说的no pubkey的错误，和前面一样解决就行了
sudo apt -y install mosquitto
mosquitto -d -p 1883
```

然后传输edgecore，你的边缘端机器ip可以在边缘端运行`ifconfig`查看执行以下操作

```bash
# 云端
cd $GOPATH/src/github.com/kubeedge/kubeedge-v1.3.1/edge
scp edgecore <你的用户名>@<你的边缘端机器ip>:/home/<你的用户名>/edgecore
# 边缘端
mkdir -p /etc/kubeedge/config
./edgecore --minconfig > /etc/kubeedge/config/edgecore.yaml
```

然后在云端获取token

```bash
# 云端
kubectl get secret tokensecret -n kubeedge -oyaml
# 输出示例：
apiVersion: v1
data:
  tokendata: MjkyOGJjMDQ4MjE4YWYyODk0OWFlOGYxNjQ4ZTY5MjQzYmY5N2ZmYTMxNTRlZGZlOWQ1MWI4YTAyYmRlMzY2YS5leUpoYkdjaU9pSklVekkxTmlJc0luUjVjQ0k2SWtwWFZDSjkuZXlKbGVIQWlPakUyTVRNNE9EZzNPREY5LnAwaXIxZmlGVUlzQ3R0ZGdFTHZ5b3pmc1l4SVM3eHZ0cnZLZjNfSWIzMmM=
kind: Secret
...
# 获取输出之后，将其中的token解码
echo MjkyOGJjMDQ4MjE4YWYyODk0OWFlOGYxNjQ4ZTY5MjQzYmY5N2ZmYTMxNTRlZGZlOWQ1MWI4YTAyYmRlMzY2YS5leUpoYkdjaU9pSklVekkxTmlJc0luUjVjQ0k2SWtwWFZDSjkuZXlKbGVIQWlPakUyTVRNNE9EZzNPREY5LnAwaXIxZmlGVUlzQ3R0ZGdFTHZ5b3pmc1l4SVM3eHZ0cnZLZjNfSWIzMmM= | base64 -d 
# 将得到的输出拷贝下来
```

然后在边缘端修改edgecore.yaml

```bash
# 边缘端
sudo vim /etc/kubeedge/config/cloudcore.yaml 
```

修改其中三个部分

修改edgehub下的token，为云端获取的token值
修改websocket下的server，以及httpServer，都需把ip改为实际云端 IP 地址。
修改（确认）podSandboxImage，X86平台为podSandboxImage: kubeedge/pause:3.1（默认），ARM 平台根据位数不同，可设为kubeedge/pause-arm:3.1或ubeedge/pause-arm64:3.1。

然后运行`./edgecore`，或者和前面一样用

```bash
# 边缘端
sudo su
nohup ./edgecore > edgecore.log 2>&1 &
```

将其挂起在后台运行。

过一段时间后用`kubectl get nodes`查看，如果看到两个node都是ready状态，那么说明集群配置成功了。

如果要加入第三个节点，按照同样的边缘端配置过程在第三个节点上运行即可。

