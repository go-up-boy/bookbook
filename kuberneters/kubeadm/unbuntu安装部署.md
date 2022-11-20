# kubeadm安装部署

系统：linux ubuntu 20.14

## 安装 kubeadm
1.配置国内镜像源
```shell
sudo apt install -y apt-transport-https ca-certificates curl

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

sudo apt update
``` 
2.获取 kubeadm、kubelet 和 kubectl 工具,这里都使用 1.23.3 版本
```shell
sudo apt install -y kubeadm=1.23.3-00 kubelet=1.23.3-00 kubectl=1.23.3-00
```

3.验证是否安装成功
```shell
kubeadm version
kubectl version --client
```
4.防止版本变动，先锁定这三个软件版本， 避免意外升级导致版本错误
```shell
sudo apt-mark hold kubeadm kubelet kubectl
```
### 下载 Kubernetes 组件镜像
1.使用命令 kubeadm config images list 可以查看安装 Kubernetes 所需的镜像列表，参数 --kubernetes-version 可以指定版本号:
```shell
kubeadm config images list --kubernetes-version v1.23.3

k8s.gcr.io/kube-apiserver:v1.23.3
k8s.gcr.io/kube-controller-manager:v1.23.3
k8s.gcr.io/kube-scheduler:v1.23.3
k8s.gcr.io/kube-proxy:v1.23.3
k8s.gcr.io/pause:3.6
k8s.gcr.io/etcd:3.5.1-0
k8s.gcr.io/coredns/coredns:v1.8.6
```
2.这里使用国内镜像下载<br>
国内镜像源有很多，推荐用 "registry.aliyuncs.com/google_containers"<br>
示例上用我的私人镜像仓库 "registry.cn-hangzhou.aliyuncs.com/hongz"
```shell
repo=registry.cn-hangzhou.aliyuncs.com/hongz

for name in `kubeadm config images list --kubernetes-version v1.23.3`; do

    src_name=${name#k8s.gcr.io/}
    src_name=${src_name#coredns/}

    docker pull $repo/$src_name

    docker tag $repo/$src_name $name
    docker rmi $repo/$src_name
done
```

### 安装Master节点 （ worker节点无需关注 ）
1.安装命令很简单，会有很多配置参数可用 -h 查看
* --pod-network-cidr，设置集群里 Pod 的 IP 地址段
* --apiserver-advertise-address，设置 apiserver 的 IP 地址，对于多网卡服务器来说很重要（比如 VirtualBox 虚拟机就用了两块网卡），可以指定 apiserver 在哪个网卡上对外提供服务。
* --kubernetes-version，指定 Kubernetes 的版本号。
* 注意 --apiserver-advertise-address 这个命令 需要填写本机的Ip
```shell
sudo kubeadm init \
    --pod-network-cidr=10.10.0.0/16 \
    --apiserver-advertise-address=172.17.0.2 \
    --kubernetes-version=v1.23.3
```
2.通常会有以下提示，按照提示执行命令即可
```shell

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
3.务必备份这个命令，work节点加入时候要用到
```shell

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.10.210:6443 --token tv9mkx.tw7it9vphe158e74 \
  --discovery-token-ca-cert-hash sha256:e8721b8630d5b562e23c010c70559a6d3084f629abad6a2920e87855f8fb96f3
```
4.此时就可以查看Node了
```shell
kubectl version
kubectl get node
```
### 安装Flannel网络插件 （Worker节点必须安装）
1.安装这个很简单，只需要使用项目的“kube-flannel.yml”在 Kubernetes 里部署一下就好了。不过因为它应用了 Kubernetes 的网段地址，你需要修改文件里的“net-conf.json”字段，把 Network 改成刚才 kubeadm 的参数 --pod-network-cidr 设置的地址段。
<br>
仓库地址：https://github.com/flannel-io/flannel/
<br>
https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
将yaml文件下载下来替换 Network地址即可
```shell
  net-conf.json: |
    {
      "Network": "10.10.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
```
2.安装网络插件
```shell
kubectl apply -f kube-flannel.yml
```

### Worker节点加入
1. 除了Master节点 的安装配置完成后只需要执行 Join命令即可，执行完 等待1分钟左右即可
```shell
kubeadm join 192.168.10.210:6443 --token tv9mkx.tw7it9vphe158e74 \
  --discovery-token-ca-cert-hash sha256:e8721b8630d5b562e23c010c70559a6d3084f629abad6a2920e87855f8fb96f3
```

### 常见错误
**network: open /run/flannel/subnet.env: no such file or directory**
<br>
在每个节点创建文件/run/flannel/subnet.env写入以下内容。注意每个节点都要加哦，不是主节点
```shell
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```
**network: failed to set bridge addr: "cni0" already has an IP address different from xxx**
查看 cni0 和 flannel.1 网卡Ip 发现两张网卡Ip不一样
<br>
我们使用的Overlay network为Flannel，也就是说Pod的IP地址段应该在Flannel的subnet下，而现在我们看到cni0的IP地址段与flannel subnet地址段不同，所以就出现了问题。
<br>
* 方法1 修改 cni0网卡 ip 和 flannel.1 一致
* 方法2 将这个错误的网卡删除掉，之后会自动重建
```shell
ifconfig cni0 down    
ip link delete cni0
```
