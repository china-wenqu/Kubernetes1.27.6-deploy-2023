[TOC]

## Kubernetes服务器集群部署

### 1、节点配置

| 主机名                   | IP            | 角色    | 系统                  | CPU/内存 | 磁盘 |
| :----------------------- | :------------ | :------ | :-------------------- | :------- | :--- |
| kubernetes-master        | 192.168.2.160 | Master  | Ubuntu Server 22.04.3 | 4 核 8G  | 1T   |
| kubernetes-node-01       | 192.168.2.161 | Node    | Ubuntu Server 22.04.3 | 4核 8G   | 1T   |
| kubernetes-node-02       | 192.168.2.162 | Node    | Ubuntu Server 22.04.3 | 4 核 8G  | 1T   |
| kubernetes-node-03       | 192.168.2.163 | Node    | Ubuntu Server 22.04.3 | 4 核 8G  | 1T   |
| kubernetes-volumes       | 192.168.2.168 | Volumes | Ubuntu Server 22.04.3 | 4 核 8G  | 1T   |
| kubernetes-volumes-mount | 192.168.2.169 | Volumes | Ubuntu Server 22.04.3 | 4 核 8G  | 1T   |

### 2、环境配置

vim粘贴格式变乱处理

```bash
:set paste
进入paste模式以后，可以在插入模式下粘贴内容，不会有任何变形 
```

root权限

```bash
# 设置root密码
sudo passwd root
# 更换root权限
su root
# 设置root远程登录
sudo vim /etc/ssh/sshd_config
# 将此行PermitRootLogin注释去掉，并设置为yes
```

开启root远程访问

```bash
# 开启root远程访问
sudo vim /etc/ssh/sshd_config

# PermitRootLogin yes
PermitRootLogin yes

# 重启SSH
systemctl restart ssh
```

网络配置

```bash
# /etc/netplan
sudo vim /etc/netplan/00-installer-config.yaml
# 静态指定
network:
  ethernets:
    ens32:
      dhcp4: false
      addresses: [192.168.2.160/24]
      routes:
        - to: default
          via: 192.168.2.1
      nameservers:
        addresses: [114.114.114.114,8.8.8.8]
  version: 2

 # 保存生效
sudo netplan apply
```

配置主机名

```bash
# 修改主机名
# master主机修改192.168.2.160
hostnamectl set-hostname kubernetes-master
# node-01主机修改192.168.2.161
hostnamectl set-hostname kubernetes-node-01
# node-02主机修改192.168.2.162
hostnamectl set-hostname kubernetes-node-02
# node-03主机修改192.168.2.163
hostnamectl set-hostname kubernetes-node-03
# kubernetes-volumes主机修改192.168.2.168
hostnamectl set-hostname kubernetes-volumes
# kubernetes-volumes-mount主机修改192.168.2.169
hostnamectl set-hostname kubernetes-volumes-mount


```

在Master主机中修改hosts

```bash
vim /etc/hosts
# 添加
192.168.2.160 kubernetes-master
192.168.2.161 kubernetes-node-01
192.168.2.162 kubernetes-node-02
192.168.2.163 kubernetes-node-03
```

配置DNS

```bash
# 取消 DNS 行注释，并增加 DNS 配置如：114.114.114.114，修改后重启下计算机
vim /etc/systemd/resolved.conf

DNS=114.114.114.114 8.8.8.8
```

关闭防火墙

```bash
配置主机名ufw disable
```

关闭交换空间

```bash
# 临时
swapoff -a  
# 永久
sed -ri 's/.*swap.*/#&/' /etc/fstab    

free -h
```



修改 cloud.cfg

主要作用是防止重启后主机名还原

```bash
vim /etc/cloud/cloud.cfg
# 该配置默认为 false，修改为 true 即可
preserve_hostname: true
```

同步时间

```bash
# 设置时区
dpkg-reconfigure tzdata
# 选择 Asia（亚洲）->Shanghai（上海）
# 时间同步
# 安装 ntpdate
apt-get install ntpdate
# 设置系统时间与网络时间同步（cn.pool.ntp.org 位于中国的公共 NTP 服务器）
ntpdate cn.pool.ntp.org
# 将系统时间写入硬件时间
hwclock --systohc
# 确认时间
date
# 输出如下（自行对照与系统时间是否一致）
Sun Jun  2 22:02:35 CST 2023
```

安装JDK

```bash
# 更新
apt-get update
# 查看jdk可安装版本
java -version
# 安装JDK
apt install openjdk-17-jre-headless
# 查看jdk是否安装成功
java -version
```

docker安装

```bash
# 卸载旧版本
sudo apt-get remove docker docker-engine docker.io containerd
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
#工具
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# 安装官方GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 更新软件源
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
#更新
sudo apt-get update

# 安装
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 加速器配置修改 `cgroup` 驱动为 `systemd`，满足 K8S 建议）
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
         "https://iqgum0ag.mirror.aliyuncs.com"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl status docker


#docker运行出现错误，我们可以使用初始化命令
wget -qO- https://get.docker.com/ | sh

```



### 3、安装容器运行时

转发 IPv4 并让 iptables 看到桥接流量

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system


#过运行以下指令确认 br_netfilter 和 overlay 模块被加载
lsmod | grep br_netfilter
lsmod | grep overlay

#通过运行以下指令确认 net.bridge.bridge-nf-call-iptables、net.bridge.bridge-nf-call-ip6tables 和 net.ipv4.ip_forward 系统变量在你的 sysctl 配置中被设置为 1
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```



所有节点上安装 Containerd

```bash
# 安装
apt-get update
apt-get upgrade

#github链接地址：https://github.com/containerd/containerd/releases
# 查看可按照版本 ubuntu在线仓库版本不是最新，可以使用github仓库中的新版本，使用二进制方式部署
apt-cache madison containerd

# 添加一个服务: sudo update-rc.d ServiceName defaults
# 删除一个服务: sudo update-rc.d ServiceName remove

# 安装
apt -y install containerd

# 确认服务启动
systemctl daemon-reload
systemctl status containerd
systemctl restart containerd

containerd --version
ctr version


mkdir /etc/containerd/
containerd config default > /etc/containerd/config.toml
# 配置 containerd 用systemdcgroup启动.

registry.k8s.io/pause 改为 registry.aliyuncs.com/google_containers/pause
SystemdCgroup=false 改为 SystemdCgroup=true


sed -i "s#registry.k8s.io/pause#registry.aliyuncs.com/google_containers/pause#g" /etc/containerd/config.toml
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml


systemctl restart containerd




# 解决CNI问题
#下载链接：https://github.com/containernetworking/plugins/releases
mkdir /usr/local/kubernetes
cd /usr/local/kubernetes
wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz
#覆盖原有的文件
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.3.0.tgz
```



### 4、安装 kubeadm、kubelet 和 kubectl

所有节点配置

1. 更新 `apt` 包索引并安装使用 Kubernetes `apt` 仓库所需要的包：

   ```shell
   sudo apt-get update
   sudo apt-get install -y apt-transport-https ca-certificates curl
   ```

1. 下载 Google Cloud 公开签名秘钥：

   ```shell
   curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
   ```
   
1. 添加 Kubernetes `apt` 仓库：

   ```shell
   echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```
   
1. 更新 `apt` 包索引，安装 kubelet、kubeadm 和 kubectl，并锁定其版本：

   ```shell
   # 查看k8s版本号
   apt-cache madison kubelet
   
   sudo apt-get update
   # 最新版本安装
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   
   # 指定版本安装
   apt-get install kubeadm=1.27.6-00 kubectl=1.27.6-00 kubelet=1.27.6-00
   sudo apt-mark hold kubelet kubeadm kubectl
   
   systemctl status kubelet
   systemctl start kubelet
   systemctl enable kubelet
   journalctl -xeu kubelet | grep Failed
   
   ```



init配置

```bash
--apiserver-advertise-address string   设置 apiserver 绑定的 IP.
--apiserver-bind-port int32            设置apiserver 监听的端口. (默认 6443)
--apiserver-cert-extra-sans strings    api证书中指定额外的Subject Alternative Names (SANs) 可以是IP 也可以是DNS名称。 证书是和SAN绑定的。
--cert-dir string                      证书存放的目录 (默认 "/etc/kubernetes/pki")
--certificate-key string                kubeadm-cert secret 中 用于加密 control-plane 证书的key
--config string                         kubeadm 配置文件的路径.
--cri-socket string                    CRI socket 文件路径，如果为空 kubeadm 将自动发现相关的socket文件; 只有当机器中存在多个 CRI  socket 或者 存在非标准 CRI socket 时才指定.
--dry-run                              测试，并不真正执行;输出运行后的结果.
--feature-gates string                 指定启用哪些额外的feature 使用 key=value 对的形式。
--help  -h                             帮助文档
--ignore-preflight-errors strings       忽略前置检查错误，被忽略的错误将被显示为警告. 例子: 'IsPrivilegedUser,Swap'. Value 'all' ignores errors from all checks.
--image-repository string              选择拉取 control plane images 的镜像repo (default "k8s.gcr.io")
--kubernetes-version string            选择K8S版本. (default "stable-1")
--node-name string                     指定node的名称，默认使用 node 的 hostname.
--pod-network-cidr string              指定 pod 的网络， control plane 会自动将 网络发布到其他节点的node，让其上启动的容器使用此网络
--service-cidr string                  指定service 的IP 范围. (default "10.96.0.0/12")
--service-dns-domain string            指定 service 的 dns 后缀, e.g. "myorg.internal". (default "cluster.local")
--skip-certificate-key-print            不打印 control-plane 用于加密证书的key.
--skip-phases strings                  跳过指定的阶段（phase）
--skip-token-print                     不打印 kubeadm init 生成的 default bootstrap token 
--token string                         指定 node 和control plane 之间，简历双向认证的token ，格式为 [a-z0-9]{6}\.[a-z0-9]{16} - e.g. abcdef.0123456789abcdef
--token-ttl duration                   token 自动删除的时间间隔。 (e.g. 1s, 2m, 3h). 如果设置为 '0', token 永不过期 (default 24h0m0s)
--upload-certs                         上传 control-plane 证书到 kubeadm-certs Secret.

```

Master创建

```bash
# 导出配置文件
cd /etc/kubernetes
kubeadm config print init-defaults --kubeconfig ClusterConfiguration > kubeadm.yml
# 修改配置文件  

apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  # 修改为主节点 IP
  advertiseAddress: 192.168.2.160
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  # 修改主节点名称
  name: kubernetes-master
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
# 国内不能访问 Google，修改为阿里云
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
# 修改版本号 与kubeadm版本相同
kubernetesVersion: v1.27.6
networking:
  dnsDomain: cluster.local
  # 配置 POD 所在网段为我们虚拟机不重叠的网段（这里用的是 Flannel 默认网段）
  podSubnet: "10.244.0.0/16"
  serviceSubnet: 10.96.0.0/12
scheduler: {}


# 拉取镜像
# 我遇到coredns拉取不下来的
#用 docker pull coredns/coredns:1.8.0 拉取
#再更改标签 docker tag coredns/coredns:1.8.0 registry.aliyuncs.com/google_containers/coredns/coredns:v1.8.0
kubeadm config images pull --config kubeadm.yml
#kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.26.7

# 安装主节点

kubeadm init --apiserver-advertise-address=192.168.2.160 --kubernetes-version v1.27.6 --image-repository registry.aliyuncs.com/google_containers  --pod-network-cidr 10.244.0.0/16 --cri-socket unix:/run/containerd/containerd.sock --v 5 --ignore-preflight-errors=Swap

  
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=/etc/kubernetes/admin.conf     
      
kubeadm join 192.168.2.160:6443 --token g7369g.ogiac3gl0bc0beny \
	--discovery-token-ca-cert-hash sha256:cbd07dee332cb601932e7e3f1b932e5d6b391885474ec2e1945faacbda9cfc55 

#重置
sudo kubeadm reset
rm -rf .kube/
sudo rm -rf /etc/kubernetes/
sudo rm -rf /var/lib/kubelet/
sudo rm -rf /var/lib/etcd


systemctl daemon-reload
systemctl restart kubelet

journalctl -xefu kubelet
```



Node加入

```bash

kubeadm join 192.168.2.160:6443 --token g7369g.ogiac3gl0bc0beny \
	--discovery-token-ca-cert-hash sha256:cbd07dee332cb601932e7e3f1b932e5d6b391885474ec2e1945faacbda9cfc55 


#默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：（主节点）
kubeadm token create --print-join-command
```



flannel网络连接组件

```bash
# 第一种方法：直接安装
# 在第一个 master 节点配置网络组件
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml


mkdir flannel
cd flannel
# 下载配置文件
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
# 部署
kubectl apply -f kube-flannel.yaml

#查看nodes
kubectl get node
kubectl get pods -n kube-flannel -o wide
# STATUS都是Ready 因为网络原因，这个过程比较慢

#所有节点执行
mkdir -p /opt/cni/bin
curl -O -L https://github.com/containernetworking/plugins/releases/download/v1.2.0/cni-plugins-linux-amd64-v1.3.0.tgz
tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-v1.3.0.tgz

# 方法二 将flannel上传到个人镜像仓库中
#主节点执行
#查看所需镜像
grep image kube-flannel.yml
#docker拉取镜像
docker pull docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.2.0
docker pull docker.io/rancher/mirrored-flannelcni-flannel:v0.22.3
#将镜像推送到Registry
$ docker login --username=chin*****@163.com registry.cn-hangzhou.aliyuncs.com
$ docker tag [ImageId] registry.cn-hangzhou.aliyuncs.com/wenqu/flannel-cni-plugin:[镜像版本号]
$ docker push registry.cn-hangzhou.aliyuncs.com/wenqu/flannel-cni-plugin:[镜像版本号]
#修改 kube-flannel.yml
image: registry.cn-hangzhou.aliyuncs.com/wenqu/flannel-cni-plugin:1.2.0
image: registry.cn-hangzhou.aliyuncs.com/wenqu/flannel:0.22.3
#执行
kubectl create -f kube-flannel.yml
```



### 5、资源NFS服务部署

| 主机名                   | IP            | 角色    | 系统                  | CPU/内存 | 磁盘 |
| :----------------------- | :------------ | :------ | :-------------------- | :------- | :--- |
| kubernetes-volumes       | 192.168.2.168 | Volumes | Ubuntu Server 22.04.3 | 4 核 8G  | 1T   |
| kubernetes-volumes-mount | 192.168.2.169 | Volumes | Ubuntu Server 22.04.3 | 4 核 8G  | 1T   |

服务端kubernetes-volumes

```bash
# 创建一个目录作为共享文件目录
mkdir -p /usr/local/kubernetes/volumes
# 给目录增加读写权限
chmod a+rw /usr/local/kubernetes/volumes
# 安装 NFS 服务端
apt-get update

apt install nfs-common nfs-kernel-server -y
```

配置 NFS 服务目录，打开文件

```bash
vi /etc/exports
# 在尾部新增一行，内容如下
/usr/local/kubernetes/volumes *(rw,sync,no_subtree_check,no_root_squash)
# 重启服务，使配置生效
/etc/init.d/nfs-kernel-server restart
```

客户端kubernetes-volumes-mount

master node01 node02 node03 volumes-mount需要安装apt-get install -y nfs-common

```bash
apt-get update
apt-get install -y nfs-common
# 创建 NFS 客户端挂载目录
mkdir -p /usr/local/kubernetes/volumes-mount
chmod a+rw /usr/local/kubernetes/volumes-mount
# 将 NFS 服务器的 `/usr/local/kubernetes/volumes` 目录挂载到 NFS 客户端的 `/usr/local/kubernetes/volumes-mount` 目录
mount 192.168.2.168:/usr/local/kubernetes/volumes /usr/local/kubernetes/volumes-mount

```

取消 NFS 客户端挂载

> **注意：** 不要直接在挂载目录下执行，否则会报错

```
umount /usr/local/kubernetes/volumes-mount
```



### 6、kubectl 常用命令介绍

```bash
#查看
kubectl get node -o wide  #获取node list
kubectl get pod -o wide   #获取pod list
kubectl get service -o wide #获取service list
kubectl get pv -o wide #获取service list
kubectl get pvc -o wide #获取service list
kubectl get namespace #获取namespace list
kubectl get deployment -o wide #获取deploy list

kubectl get pod -n namespace

#删除
kubectl delete service 名称 #删除service
kubectl delete pod -n namespace 名称 #删除pod
kubectl delete namespace 名称 #删除namespace
kubectl delete pv 名称 #删除pv
kubectl delete pvc 名称 #删除pvc

#创建
kubectl apply -f xx.yaml #部署服务、部署service
kubectl create -f xx.yaml

#详情
kubectl describe node loadtest-worker1 #获取node详细信息
kubectl describe pod kube-flannel-ds-n9mf9 -n kube-system #查看pod详细信息  -n后面是namespace

#日志
kubectl logs -f tomcat-deployment-84ff9bf6b4-54bdp #查看日志

#进入容器
kubectl exec -it pod /bin/bash #进入容器
```



### 7、服务部署

pv  pvc configMap deployment  service  

#### 部署mysql

```bash
#创建namespace
kubectl create namespace mysql-cluster
```



创建持久卷

kubectl create -f nfs-pv-mysql.yaml -n mysql-cluster

```bash
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-mysql
spec:
  # 设置容量
  capacity:
    storage: 10Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/volumes/mysql/data"
    # NFS 服务端地址
    server: 192.168.2.168
    readOnly: false


```



```bash
# 部署
kubectl create -f nfs-pv-mysql.yaml -n mysql-cluster
# 删除
kubectl delete -f nfs-pv-mysql.yaml -n mysql-cluster
# 查看
kubectl get pv -n mysql-cluster

```



创建持久卷声明

kubectl create -f nfs-pvc-mysql.yaml -n mysql-cluster

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-mysql
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 10Gi

```



```bash
# 部署
kubectl create -f nfs-pvc-mysql.yaml -n mysql-cluster
# 删除
kubectl delete -f nfs-pvc-mysql.yaml -n mysql-cluster
# 查看
kubectl get pvc -n mysql-cluster
```



kubectl create -f mysql-config.yaml -n mysql-cluster

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-cluster-config
data:
  # 这里是键值对数据
  mysqld.cnf: |
    [client]
    port=3306
    [mysql]
    no-auto-rehash
    [mysqld]
    skip-host-cache
    skip-name-resolve
    default-authentication-plugin=mysql_native_password
    character-set-server=utf8mb4
    collation-server=utf8mb4_general_ci
    explicit_defaults_for_timestamp=true
    lower_case_table_names=1
```

kubectl create -f mysql-deployment.yaml -n mysql-cluster



```bash
#给k8s-node1 k8s-node2打上标签mysql-cluster
$ kubectl label nodes kubernetes-node-01 kubernetes-node-02 type=mysql-cluster

#查看type=mysql-cluster标签的node
$ kubectl get node -l type=mysql-cluster

#以下附带标签的其他操作：
#修改标签
$ kubectl label nodes kubernetes-node01 kubernetes-node02 type=test --overwrite
#查看node标签
$ kubectl get nodes kubernetes-node01 kubernetes-node02 --show-labels
#删除标签
$ kubectl label nodes kubernetes-node01 kubernetes-node02 type-


#mysql-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-cluster
spec:
  selector:
    matchLabels:
      name: mysql-cluster
  replicas: 2
  template:
    metadata:
      labels:
        name: mysql-cluster
    spec:
      nodeSelector:
        type: mysql-cluster
      containers:
        - name: mysql-cluster
          image: mysql:8.0.34
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "123456"
          volumeMounts:
            # 以数据卷的形式挂载 MySQL 配置文件目录
            - name: cm-vol-mysql-cluster
              mountPath: /etc/mysql/conf.d
            - name: nfs-vol-mysql-cluster
              mountPath: /var/lib/mysql
      volumes:
        # 将 ConfigMap 中的内容以文件形式挂载进数据卷
        - name: cm-vol-mysql-cluster
          configMap:
            name: mysql-cluster-config
            items:
                # ConfigMap 中的 Key
              - key: mysqld.cnf
                # ConfigMap Key 匹配的 Value 写入名为 mysqld.cnf 的文件中
                path: mysqld.cnf
        - name: nfs-vol-mysql-cluster
          persistentVolumeClaim:
            claimName: nfs-pvc-mysql

```

kubectl create -f mysql-service.yaml -n mysql-cluster

```bash
apiVersion: v1
kind: Service
metadata:
  name: mysql-cluster
spec:
  ports:
    - port: 3306
      targetPort: 3306
  type: LoadBalancer
  selector:
    name: mysql-cluster
```



```
kubectl get pod  -n mysql-cluster | grep mysql
```



```bash
# 查看
kubectl get pod  -n mysql-cluster | grep mysql


kubectl get pod -n mysql-cluster -o wide
kubectl describe deployment mysql-cluster -n mysql-cluster
kubectl exec -it mysql-cluster-5c94c646c-j67jg -- /bin/bash
```



#### 部署Consul

```bash
#创建namespace
kubectl create namespace consul-cluster

#statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: consul
spec:
  serviceName: consul
  replicas: 3
  selector:
    matchLabels:
      app: consul
  template:
    metadata:
      labels:
        app: consul
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: consul
        image: hashicorp/consul:1.16.2
        args:
             - "agent"
             - "-server"
             - "-bootstrap-expect=3"
             - "-ui"
             - "-data-dir=/consul/data"
             - "-bind=0.0.0.0"
             - "-client=0.0.0.0"
             - "-advertise=$(PODIP)"
             - "-retry-join=consul-0.consul.$(NAMESPACE).svc.cluster.local"
             - "-retry-join=consul-1.consul.$(NAMESPACE).svc.cluster.local"
             - "-retry-join=consul-2.consul.$(NAMESPACE).svc.cluster.local"
             - "-domain=cluster.local"
             - "-disable-host-node-id"
        env:
            - name: PODIP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
        ports:
            - containerPort: 8500
              name: ui-port
            - containerPort: 8443
              name: https-port

#service.yaml
apiVersion: v1
kind: Service
metadata:
  name: consul
  labels:
    name: consul
spec:
  type: NodePort
  ports:
    - name: http
      port: 8500
      nodePort: 30850
      targetPort: 8500
    - name: https
      port: 8443
      nodePort: 30851
      targetPort: 8443
  selector:
    app: consul

#创建
kubectl create -f statefulset.yaml -n consul-cluster
kubectl create -f service.yaml -n consul-cluster

#查看
kubectl get pod -n consul-cluster

#控制台
192.168.2.160:30850

```

#### 部署RocketMQ

部署工具rocketmq-operator

```bash
#创建文件夹
mkdir -p /usr/local/kubernetes/rocketmq
cd /usr/local/kubernetes/rocketmq

#编译工具 需要GO环境
# 下载安装 官方文档要求 go1.16版本
wget https://studygolang.com/dl/golang/go1.16.linux-amd64.tar.gz
tar -zxvf go1.16.linux-amd64.tar.gz -C /usr/local

vi /etc/profile

export PATH=$PATH:/usr/local/go/bin

source /etc/profile
go version

# 克隆
git clone https://github.com/apache/rocketmq-operator.git
cd rocketmq-operator && make deploy

# 检验
$ kubectl get crd | grep rocketmq.apache.org
brokers.rocketmq.apache.org                 2022-05-12T09:23:18Z
consoles.rocketmq.apache.org                2022-05-12T09:23:19Z
nameservices.rocketmq.apache.org            2022-05-12T09:23:18Z
topictransfers.rocketmq.apache.org          2022-05-12T09:23:19Z

### check whether operator is running
$ kubectl get pods | grep rocketmq-operator
rocketmq-operator-6f65c77c49-8hwmj   1/1     Running   0          93s

#删除
make undeploy

# 强制删除
kubectl delete pod rocketmq-operator-6759778bd9-pww7z --force --grace-period=0

```



##### 1、直接安装

```bash
# 执行
cd example && kubectl create -f rocketmq_v1alpha1_rocketmq_cluster.yaml
# 检验
kubectl get sts
NAME                 READY   AGE
broker-0-master      1/1     107m
broker-0-replica-1   1/1     107m
name-service         1/1     107m
# 控制端发布
# rocketmq-console-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: rocketmq-service
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 32568
  selector:
    app: rocketmq-console
    
 #执行
 kubectl apply -f rocketmq-console-service.yaml
 
# 检验
# 访问您集群中任意节点的 32568 端口
http://192.168.2.160:32568
```

##### 2、自定义安装

部署数据卷

kubernetes-volumes      

```bash
mkdir -p /usr/local/kubernetes/volumes/rocketmq/data
chmod a+rw /usr/local/kubernetes/volumes/rocketmq/data

vi /etc/exports
# 底部增加
/usr/local/kubernetes/volumes/rocketmq/data *(rw,sync,no_subtree_check,no_root_squash)
# 重启服务，使配置生效
/etc/init.d/nfs-kernel-server restart
```



部署StorageClass

新建 rocketmq-pv.yaml

部署三个PV

```bash
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: namesrv-storage
spec:
  # 设置容量
  capacity:
    storage: 10Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/volumes/rocketmq/data/namesrv"
    # NFS 服务端地址
    server: 192.168.2.168
    readOnly: false
  storageClassName: rocketmq-storage
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: broker-storage-master-0
spec:
  # 设置容量
  capacity:
    storage: 10Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/volumes/rocketmq/data/broker/master/0"
    # NFS 服务端地址
    server: 192.168.2.168
    readOnly: false
  storageClassName: rocketmq-storage
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: broker-storage-replica-0
spec:
  # 设置容量
  capacity:
    storage: 10Gi
  # 访问模式
  accessModes:
    # 该卷能够以读写模式被多个节点同时加载
    - ReadWriteMany
  # 回收策略，这里是基础擦除 `rm-rf/thevolume/*`
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    # NFS 服务端配置的路径
    path: "/usr/local/kubernetes/volumes/rocketmq/data/broker/replica/0"
    # NFS 服务端地址
    server: 192.168.2.168
    readOnly: false
  storageClassName: rocketmq-storage
 

```

新建 rocketmq-pvc.yaml

部署三个PVC

```bash
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: namesrv-storage-name-service-0
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 10Gi
  storageClassName: rocketmq-storage
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broker-storage-broker-0-master-0
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 10Gi
  storageClassName: rocketmq-storage
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: broker-storage-broker-0-replica-1-0
spec:
  accessModes:
  # 需要使用和 PV 一致的访问模式
  - ReadWriteMany
  # 按需分配资源
  resources:
     requests:
       storage: 10Gi
  storageClassName: rocketmq-storage
```





```bash
kubectl create -f rocketmq-pv.yaml

kubectl create -f  rocketmq-pvc.yaml

kubectl get pvc
NAME                                  STATUS   VOLUME                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
broker-storage-broker-0-master-0      Bound    broker-storage-master-0    10Gi       RWX            rocketmq-storage   65m
broker-storage-broker-0-replica-1-0   Bound    broker-storage-replica-0   10Gi       RWX            rocketmq-storage   65m
namesrv-storage-name-service-0        Bound    namesrv-storage            10Gi       RWX            rocketmq-storage   65m

kubectl get pv
NAME                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                         STORAGECLASS       REASON   AGE
broker-storage-master-0    10Gi       RWX            Recycle          Bound    default/broker-storage-broker-0-master-0      rocketmq-storage            68m
broker-storage-replica-0   10Gi       RWX            Recycle          Bound    default/broker-storage-broker-0-replica-1-0   rocketmq-storage            68m
namesrv-storage            10Gi       RWX            Recycle          Bound    default/namesrv-storage-name-service-0        rocketmq-storage            68m
```



kubernetes-master   

修改配置文件nfs-client.yaml

/usr/local/kubernetes/rocketmq/rocketmq-operator/deploy/storage/nfs-client.yaml

```bash
vi deploy/storage/nfs-client.yaml

kind: Deployment
# 修改apiVersion
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  # 添加标签
  selector: 
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-:wqprovisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: rocketmq/nfs
            - name: NFS_SERVER
              # 挂载IP
              value: 192.168.2.168
            - name: NFS_PATH
              # 挂载地址
              value: /usr/local/kubernetes/volumes/rocketmq/data
      volumes:
        - name: ku
          nfs:
            server: 192.168.2.168
            path: /usr/local/kubernetes/volumes/rocketmq/data

```

创建 StorageClass

```bash
cd deploy/storage/
./deploy-storage-class.sh

# 检验
kubectl get pods

NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-7487d8b858-mvzkh   1/1     Running   0          3m42s
rocketmq-operator-6759778bd9-59knz        1/1     Running   0          3h56m
```

部署Cluster

部署NameServer

修改配置文件 rocketmq_v1alpha1_nameservice_cr.yaml

/usr/local/kubernetes/rocketmq/rocketmq-operator/example/rocketmq_v1alpha1_nameservice_cr.yaml

```bash
vi example/rocketmq_v1alpha1_nameservice_cr.yaml


apiVersion: rocketmq.apache.org/v1alpha1
kind: NameService
metadata:
  name: name-service
  namespace: default
spec:
  # size is the the name service instance number of the name service cluster
  size: 1
  # nameServiceImage is the customized docker image repo of the RocketMQ name service
  nameServiceImage: apacherocketmq/rocketmq-nameserver:4.5.0-alpine-operator-0.3.0
  # imagePullPolicy is the image pull policy
  imagePullPolicy: Always
  # hostNetwork can be true or false
  hostNetwork: true
  #  Set DNS policy for the pod.
  #  Defaults to "ClusterFirst".
  #  Valid values are 'ClusterFirstWithHostNet', 'ClusterFirst', 'Default' or 'None'.
  #  DNS parameters given in DNSConfig will be merged with the policy selected with DNSPolicy.
  #  To have DNS options set along with hostNetwork, you have to specify DNS policy
  #  explicitly to 'ClusterFirstWithHostNet'.
  dnsPolicy: ClusterFirstWithHostNet
  # resources describes the compute resource requirements and limits
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1024Mi"
      cpu: "500m"
  # storageMode can be EmptyDir, HostPath, StorageClass
  # 类型选择StorageClass
  storageMode: StorageClass
  # hostPath is the local path to store data
  hostPath: /data/rocketmq/nameserver
  # volumeClaimTemplates defines the storageClass
  volumeClaimTemplates:
    - metadata:
        name: namesrv-storage
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: rocketmq-storage
        resources:
          requests:
            storage: 1Gi

```

部署rocketmq_v1alpha1_nameservice_cr.yaml

```bash
kubectl apply -f example/rocketmq_v1alpha1_nameservice_cr.yaml
# 输出如下
nameservice.rocketmq.apache.org/name-service created
#检验 记住这个IP地址192.168.2.163 后面部署要用到
kubectl get pods -o wide
NAME                                     READY   STATUS                   RESTARTS   AGE     IP              NODE                 NOMINATED NODE   READINESS GATES
name-service-0                           1/1     Running                  0          3m19s   192.168.2.163   kubernetes-node-03   <none>           <none>

```

部署Broker


```bash
vi example/rocketmq_v1alpha1_broker_cr.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: broker-config
data:
  BROKER_MEM: " -Xms2g -Xmx2g -Xmn1g "
  broker-common.conf: |
    # brokerClusterName, brokerName, brokerId are automatically generated by the operator and do not set it manually!!!
    deleteWhen=04
    fileReservedTime=48
    flushDiskType=ASYNC_FLUSH
    # set brokerRole to ASYNC_MASTER or SYNC_MASTER. DO NOT set to SLAVE because the replica instance will automatically be set!!!
    brokerRole=ASYNC_MASTER

---
apiVersion: rocketmq.apache.org/v1alpha1
kind: Broker
metadata:
  # name of broker cluster
  name: broker
spec:
  # size is the number of the broker cluster, each broker cluster contains a master broker and [replicaPerGroup] replica brokers.
  size: 1
  # nameServers is the [ip:port] list of name service
  # 修改为你部署的 name-service-x 的地址
  # kubectl get pod -o wide 查看IP
  nameServers: 192.168.2.162:9876
  # replicaPerGroup is the number of each broker cluster
  replicaPerGroup: 1
  # brokerImage is the customized docker image repo of the RocketMQ broker
  brokerImage: registry.cn-hangzhou.aliyuncs.com/wenqu/rocketmq-broker:4.5.0-alpine-operator-0.3.0
  # imagePullPolicy is the image pull policy
  imagePullPolicy: Always
  # resources describes the compute resource requirements and limits
  resources:
    requests:
      memory: "2048Mi"
      cpu: "250m"
    limits:
      memory: "12288Mi"
      cpu: "500m"
  # allowRestart defines whether allow pod restart
  allowRestart: true
  # storageMode can be EmptyDir, HostPath, StorageClass
  # 类型选择StorageClass
  storageMode: StorageClass
  # hostPath is the local path to store data
  hostPath: /data/rocketmq/broker
  # scalePodName is [Broker name]-[broker group number]-master-0
  scalePodName: broker-0-master-0
  # env defines custom env, e.g. BROKER_MEM
  env:
    - name: BROKER_MEM
      valueFrom:
        configMapKeyRef:
          name: broker-config
          key: BROKER_MEM
  # volumes defines the broker.conf
  volumes:
    - name: broker-config
      configMap:
        name: broker-config
        items:
          - key: broker-common.conf
            path: broker-common.conf
  # volumeClaimTemplates defines the storageClass
  volumeClaimTemplates:
    - metadata:
        name: broker-storage
        annotations:
          volume.beta.kubernetes.io/storage-class: rocketmq-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 8Gi


```

部署rocketmq_v1alpha1_broker_cr.yaml

/usr/local/kubernetes/rocketmq/rocketmq-operator/example/rocketmq_v1alpha1_broker_cr.yaml

```bash
kubectl apply -f example/rocketmq_v1alpha1_broker_cr.yaml
# 输出如下
broker.rocketmq.apache.org/broker created
# 检验
kubectl get pods -owide
```

部署控制台

rocketmq_v1alpha1_console_cr.yaml

/usr/local/kubernetes/rocketmq/rocketmq-operator/example/rocketmq_v1alpha1_console_cr.yaml

```bash
apiVersion: rocketmq.apache.org/v1alpha1
kind: Console
metadata:
  name: console
  namespace: default
spec:
  # nameServers is the [ip:port] list of name service
  # 修改为你部署的 name-service-x 的地址
  # kubectl get pod -o wide 查看IP
  nameServers: 192.168.2.162:9876
  # consoleDeployment define the console deployment
  consoleDeployment:
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app: rocketmq-console
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: rocketmq-console
      template:
        metadata:
          labels:
            app: rocketmq-console
        spec:
          containers:
            - name: console
              image: apacherocketmq/rocketmq-console:2.0.0
              ports:
                - containerPort: 8080

```

控制台WEB  rocketmq-console-service.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: rocketmq-service
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 32568
  selector:
    app: rocketmq-console
```



```bash
kubectl apply -f rocketmq-console-service.yaml
```

访问您集群中任意节点的 32568 端口

http://192.168.2.160:32568

删除

```bash
# 删除 Console
kubectl delete -f rocketmq-console.yaml
# 删除 Broker
kubectl delete -f example/rocketmq_v1alpha1_broker_cr.yaml
# 删除 NameServer
kubectl delete -f example/rocketmq_v1alpha1_nameservice_cr.yaml
# 删除 Operator
sh ./purge-operator.sh
# 删除 Storage
cd deploy/storage
sh ./remove-storage-class.sh
```





### 8、部署SpringBoot应用

```bash
# 安装docker
# 卸载旧版本
sudo apt-get remove docker docker-engine docker.io containerd runc
# 更新软件源
sudo apt-get update
# 安装工具
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
# 安装官方GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 源
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 再次更新
sudo apt-get update

# 安装
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 选择版本安装
# 查看版本
apt-cache madison docker-ce
# 安装
sudo apt-get install docker-ce=5:20.10.13~3-0~ubuntu-jammy docker-ce-cli=5:20.10.13~3-0~ubuntu-jammy containerd.io docker-compose-plugin

# 测试
sudo docker -v
docker run hello-world

# 安装jdk
docker pull openjdk:17

# Dockerfile
vim Dockerfile

# 该镜像需要依赖的基础镜像
FROM openjdk:17
# 将targer目录下的jar包复制到docker容器/usr/local/springboot/目录下面目录下面
ADD springboot-demo.jar /usr/local/springboot/springboot-demo.jar
# 声明服务运行在8001端口
EXPOSE 3001
# 执行命令
CMD ["java","-jar","/usr/local/springboot/springboot-demo.jar"]
# 指定维护者名称
MAINTAINER wenqu china.xsw@163.com


# 执行制作镜像
docker build -t springboot-demo -f Dockerfile .

# 查看镜像
docker images

# 注册阿里云，后台创建个人镜像仓库
# 登录阿里镜像仓库
sudo docker login --username=china.xsw@163.com registry.cn-hangzhou.aliyuncs.com

# 添加版本号  要用到镜像ID
# 命令: sudo docker tag [ImageId] [仓库地址]/[命名空间名字]/[仓库名字]:[镜像版本号]
sudo docker tag 4afc0e228df1 registry.cn-hangzhou.aliyuncs.com/wenqu/springboot-demo:1.0

# 上传镜像
# 命令:sudo docker push [仓库地址]/[命名空间名字]/[仓库名字]:[镜像版本号]
sudo docker push registry.cn-hangzhou.aliyuncs.com/wenqu/springboot-demo:1.0

# 拉取镜像
# 命令: sudo docker pull [仓库地址]/[命名空间名字]/[仓库名字]:[镜像版本号]
sudo docker pull registry.cn-hangzhou.aliyuncs.com/wenqu/springboot-demo:1.0

#deployment部署镜像创建出pod,让后再进行修改即可
# 命令: kubectl create deployment [deployment名字]–image=[镜像地址] --dry-run -o yaml >[yaml文件名].yaml

kubectl create  deployment wenqu --image=registry.cn-hangzhou.aliyuncs.com/wenqu/springboot-demo:1.0 --dry-run -o yaml >deployment.yaml


# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: springboot-demo
  labels:
   name: springboot-demo
spec:
  type: NodePort
  ports:
  - port: 3001
    targetPort: 3001
    nodePort: 3001
  selector:
    app: springboot-demo


# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: springboot-demo
  name: springboot-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: springboot-demo
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: springboot-demo
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/wenqu/springboot-demo:1.0
        name: springboot-demo
        ports:
        - containerPort: 3001
          name: springboot-demo
        resources: {}
status: {}

# 部署镜像
kubectl apply -f deployment.yaml

#查验
kubectl get pod
kubectl  get svc

# 访问
http://192.168.2.160:3001/hello
```



### 9、部署WEB项目

```bash
#Dockerfile
FROM nginx
# 删除官方nginx镜像默认的配置
RUN rm -rf /etc/nginx/conf.d/default.conf
# 将nginx.conf(自己的nginx配置)添加到默认配置目录下
# 注意：nginx.conf需要与Dockerfile在同一目录
ADD ./nginx.conf /etc/nginx/conf.d/
COPY ./build /usr/share/nginx/dist

#nginx.conf
server {
    listen       9005;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/dist;
        index  index.html index.htm;
        try_files $uri /index.html;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/dist;
    }
}

#deployment.yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: web-demo
  name: web-demo
spec:
  ports:
  # 对外暴露的端口
  - nodePort: 9005
    port: 9005
    protocol: TCP
    targetPort: 9005
  selector:
    app: web-demo
  # NodePort类型可以对外暴露端口
  type: NodePort

---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web-demo
  name: web-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-demo
  template:
    metadata:
      labels:
        app: web-demo
    spec:
      containers:
      # 镜像名称
      - image: registry.cn-hangzhou.aliyuncs.com/wenqu/web-demo:1.0
        name: web-demo
        ports:
        - containerPort: 9005
        resources: {}


#制作镜像
docker build -t web-demo .

#发送镜像
$ docker login --username=chin*****@163.com registry.cn-hangzhou.aliyuncs.com
$ docker tag [ImageId] registry.cn-hangzhou.aliyuncs.com/wenqu/web-demo:[镜像版本号]
$ docker push registry.cn-hangzhou.aliyuncs.com/wenqu/web-demo:[镜像版本号]

#执行
kubuctl create -f deployment.yaml


```

