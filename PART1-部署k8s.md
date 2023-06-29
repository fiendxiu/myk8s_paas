三个4c8g的kvm虚拟机，操作系统centos 7.8.2003，内核3.10.0-1127
- ops:192.168.122.11
- master:192.168.122.12
- worker:192.168.122.13

# 初始化虚拟机
## 设置aliyum源
```shell
sed -i 's/enabled=1/enabled=0/' /etc/yum/pluginconf.d/fastestmirror.conf
sed -i 's/plugins=1/plugins=0/' /etc/yum.conf
cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
curl -sSLo /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -sSLo /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
sed -i '/aliyuncs/d' /etc/yum.repos.d/CentOS-Base.repo
yum clean all
yum makecache
```

## 安装常用软件
```shell
yum -y install wget lsof telnet netcat net-tools bind-utils bash-completion yum-utils tcpdump bridge-utils pciutils
```

## 关闭防火墙
```shell
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
systemctl --now disable firewalld
```

## 系统日志保存方式设置
centos7以后，引导方式改为了systemd，所以会有两个日志系统同时工作只保留一个日志（journald）的方法，设置rsyslogd和systemd journald
```shell
# 手动创建journald日志持久化目录和扩展配置目录
$ mkdir /var/log/journal
$ mkdir /etc/systemd/journald.conf.d

# 编辑扩展配置文件，内容如下
$ vi /etc/systemd/journald.conf.d/99-prophet.conf

[Journal]
# 持久化保存到磁盘
Storage=persistent
# 压缩历史日志
Compress=yes
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
# 最大占用空间10G
SystemMaxUse=10G
# 单日志文件最大200M
SystemMaxFileSize=200M
# 日志保存时间2周
MaxRetentionSec=2week
# 不将日志转发到syslog
ForwardToSyslog=no

# 最后重启journald
systemctl restart systemd-journald
```



# 部署kubernetes

## k8s环境预设
### 网卡uuid
```shell
$ cat /sys/class/dmi/id/product_uuid
###
查看各个节点机器的网卡uuid，确保没有重复的。这个从没碰上会重复的情况，但官网建议做这个检查
###
```

### 加载br_netfilter模块
```shell
$ lsmod | grep br_netfilter
$ modprobe br_netfilter
###
默认时是没有加载的，所以手动加载一次。后面通过配置持久化方式每次开机自动加载
###

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF
###
持久化方式设置，每次开机都自动加载br_netfilter模块。以后相关需求也可以直接编辑这份文件
由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs的前提需要加载ip_vs相关模块
###

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
###
持久化方式设置，k8s有关的内核参数。以后相关需求也可以直接编辑这份文件
###

sysctl --system
###
上面持久化方式设置的内核参数文件，通过执行sysctl --system来即时生效
###
```

### 为第一个节点做免密
免密是可选操作，非必要的
```shell
[root@ops ~]# ssh-keygen -q -t rsa -N "" -f /root/.ssh/id_rsa
[root@ops ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub 192.168.122.12
[root@ops ~]# ssh-copy-id -i ~/.ssh/id_rsa.pub 192.168.122.13
```
kubernetes环境依赖主机名通信，比如kubeadm初始化时就依赖主机名。所以这里需要给各节点配置好hosts记录，否则kubeadm init初始化控制平面时会报错的
```shell
[root@ops ~]# vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.122.11 app11 ops
192.168.122.12 app12 master
192.168.122.13 app13 worker

[root@ops ~]# for i in 192.168.122.1{2..3};do scp /etc/hosts $i:/etc/ ; done
```

### 关闭swap
kubeadm初始化控制平面时，要求swap必须是关闭的
```shell
$ swapoff -a
$ sed -i '/swap/s/^/#/' /etc/fstab
```


## 增加kubernetes源和docker源
```shell
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
$ yum -y makecache
$ yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

## 安装docker
安装docker-ce并设置服务开机自启
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum install -y docker-ce-cli docker-ce containerd.io
systemctl --now enable docker
```
设置docker配置文件daemon.json
```shell
$ vi /etc/docker/daemon.json
{
    "registry-mirrors": ["https://mf0d5bw3.mirror.aliyuncs.com"],
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "100m"
    },
    "storage-driver": "overlay2",
    "storage-opts": [
      "overlay2.override_kernel_check=true"
    ]
}
```
重载配置文件并重启docker服务
```shell
$ systemctl daemon-reload 
$ systemctl restart docker 
```

## 配置k8s使用containerd作为cri
1.生成默认的配置文件内容添加到/etc/containerd/config.toml，注释原先的
2.对于使用systemd作为init system的Linux的发行版，使用systemd作为容器的cgroup driver可以确保服务器节点在资源紧张的情况更加稳定，因此这里配置各个节点上containerd的cgroup driver为systemd
3.修改pause镜像的版本sandbox_image = "registry.k8s.io/pause:3.9"
4.修改registy配置
```shell
$ containerd config default >> /etc/containerd/config.toml
$ vi /etc/containerd/config.toml
#disabled_plugins = ["cri"]				#默认的disabled_plugins列表注释掉
            SystemdCgroup = true	# 默认的false改为true
    sandbox_image = "registry.k8s.io/pause:3.9"		# 修改pause版本为3.9
    ##containerd配置国内镜像
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = ""
      [plugins."io.containerd.grpc.v1.cri".registry.auths]
      [plugins."io.containerd.grpc.v1.cri".registry.configs]
      [plugins."io.containerd.grpc.v1.cri".registry.headers]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://mf0d5bw3.mirror.aliyuncs.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."gcr.io"]
          endpoint = ["https://gcr.lank8s.cn"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["https://lank8s.cn"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]
          endpoint = ["https://lank8s.cn"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
          endpoint = ["https://quay.azk8s.cn"]

$ systemctl daemon-reload
$ systemctl enable --now containerd
$ systemctl restart containerd
```

## 安装kubernetes
安装kubelet并设置kubelet开机自启
```shell
$ yum install -y kubectl-1.26.4 kubelet-1.26.4 kubeadm-1.26.4
$ systemctl --now enable kubelet
```
配置kubectl命令补全功能，推荐配置在ops和master机
```shell
$ echo "source <(kubectl completion bash)" >> ~/.bash_profile
$ source ~/.bash_profile
```

## 初始化master节点
初始化要指定--pod-network-cidr参数，不然之后安装cni calico时会失败
```shell
$ kubeadm init --pod-network-cidr 172.20.122.0/24 --service-cidr 172.30.0.0/16 --kubernetes-version v1.26.4
```

## 安装cni calico
推荐用helm安装calico，在ops机器上先下载helm安装包
```shell
[root@ops ~]# tar -zxf ./helm-v3.11.3-linux-amd64.tar.gz && mv ./linux-amd64/helm /usr/local/sbin/helm && chmod a+x /usr/local/sbin/helm && rm -rf ./linux-amd64 && rm -f ./helm-v3.11.3-linux-amd64.tar.gz
```
从master机器上复制kubeconfig到ops机器上，使ops机器能够控制k8s集群
```shell
scp master:/etc/kubernetes/admin.conf /etc/kubernetes/
```
添加calico chart仓库
```shell
helm repo add projectcalico https://docs.tigera.io/calico/charts
```
创建chart的values.yaml配置文件
```shell
vi values.yaml
installation:
  cni:
    type: Calico
  calicoNetwork:
    ipPools:
    - cidr: 172.20.0.0/16
    nodeAddressAutodetectionV4:
      kubernetes: NodeInternalIP

apiServer:
  enabled: false
```
使用value.yaml配置文件部署calico
```shell
helm install calico projectcalico/tigera-operator --version v3.26.0 --namespace tigera-operator --create-namespace -f values.yaml
```

## 添加worker节点到集群
master节点初始化完成后，有一些输出信息，其中最下方的内容提示了worker节点加入集群的命令。在worker上执行kubeadm join
```shell
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.122.12:6443 --token 2vq7b0.7m4w09goc22b1i5t \
	--discovery-token-ca-cert-hash sha256:259542fcb1ee548eb562480354e2b5d5630e5c396fbdc8384240944883e7f4d3
```
如果找不到join信息了，可以再创建一个token并显示出join信息。token有效期24小时，过期后自动删除
```shell
[root@master ~]# kubeadm token create --print-join-command
kubeadm join 192.168.122.12:6443 --token 3kqa78.48olrp7fd9zazwn7 --discovery-token-ca-cert-hash sha256:259542fcb1ee548eb562480354e2b5d5630e5c396fbdc8384240944883e7f4d3 
```
