扩展ops管理能力

# Ansible
ops机使用ansible shell模块对多台机器批量执行shell。
安装ansible
```shell
[root@ops ~]# yum -y install ansible
```
在/etc/ansible/hosts文件添加目标节点
```shell
$ cat <<EOF >> /etc/ansible/hosts

[apps]   
app11
app12
app13

[k8s]   
master
worker
EOF
```
测试批量执行
```shell
[root@ops ~]# ansible k8s -m shell -a 'hostname -i'
worker | CHANGED | rc=0 >>
192.168.122.13
master | CHANGED | rc=0 >>
192.168.122.12
```

# NTP
安装chrony
```shell
[root@ops ~]# ansible apps -m shell -a 'yum install -y chrony'
```
修改chrony.conf
```shell
[root@ops ~]# ansible apps -m shell -a 'sed -i "/^server/s/^/#/" /etc/chrony.conf'
[root@ops ~]# ansible apps -m shell -a 'sed -i "/server 3/s/^.*$/server ntp1.aliyun.com iburst/" /etc/chrony.conf'
```
启动时间同步
```shell
[root@ops ~]# ansible apps -m shell -a 'timedatectl set-ntp true'
```

# Harbor
官网`https://goharbor.io/docs/2.8.0/install-config/configure-yml-file/`
## 升级ops机的openssl
安装依赖库
```shell
yum install -y zlib zlib-dev openssl-devel sqlite-devel bzip2-devel libffi libffi-devel gcc gcc-c++
```
下载openssl安装包并安装
```shell
wget --no-check-certificate http://www.openssl.org/source/openssl-1.1.1.tar.gz &&\
tar -zxvf openssl-1.1.1.tar.gz &&\
cd openssl-1.1.1 &&\
./config --prefix=/opt/openssl shared zlib &&\
make && make install &&\
cd - && rm -rf openssl-1.1.1*
```
设置环境变量LD_LIBRARY_PATH.LD_LIBRARY_PATH环境变量主要用于指定查找共享库（动态链接库）时除了默认路径之外的其他路径。当执行函数动态链接.so时，如果此文件不在缺省目录下‘/lib' and ‘/usr/lib'，那么就需要指定环境变量LD_LIBRARY_PATH
```shell
echo "export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/openssl/lib" >> $HOME/.bash_profile && source $HOME/.bash_profile
```
## 使用openssl生成自签名证书
规划harbor使用的域名，比如：reg.myk8s.vm
创建目录存放harbor证书和安装包
```shell
mkdir -p ~/harbor_install/cert && cd ~/harbor_install/cert
```
生成证书颁发机构CA证书私钥（ca.key）
```shell
openssl genrsa -out ca.key 4096
```
生成证书颁发机构CA证书（ca.crt），注意CN=reg.myk8s.vm这里要填自己使用的域名
```shell
openssl req -x509 -new -nodes -sha512 -days 3650 -subj "/C=CN/ST=Guangdong/L=Dongguan/O=indax/OU=Personal/CN=reg.myk8s.vm" -key ca.key -out ca.crt
```
生成服务器证书私钥（reg.myk8s.vm.key）
```shell
openssl genrsa -out reg.myk8s.vm.key 4096
```
生成服务器证书签名请求（CSR）（reg.myk8s.vm.csr）
```shell
openssl req -sha512 -new -subj "/C=CN/ST=Guangdong/L=Dongguan/O=indax/OU=Personal/CN=reg.myk8s.vm" -key reg.myk8s.vm.key -out reg.myk8s.vm.csr
```
生成x509 v3扩展文件
```shell
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=reg.myk8s.vm
DNS.2=reg.myk8s
DNS.3=reg
EOF
```
使用v3.ext文件为主机生成证书（reg.myk8s.vm.crt）
```shell
openssl x509 -req -sha512 -days 3650 -extfile v3.ext -CA ca.crt -CAkey ca.key -CAcreateserial -in reg.myk8s.vm.csr -out reg.myk8s.vm.crt
```
将crt转换为cert格式（reg.myk8s.vm.cert）
```shell
openssl x509 -inform PEM -in reg.myk8s.vm.crt -out reg.myk8s.vm.cert
```
最终使用到的是3个文件
```shell
├── yourdomain.com.cert  <-- Server certificate signed by CA
├── yourdomain.com.key   <-- Server key signed by CA
└── ca.crt               <-- Certificate authority that signed the registry certificate
```
## 安装Harbor
创建harbor数据目录，下载harbor离线安装包
```shell
mkdir -p /home/t4/harbor && cd ~/harbor_install/ && wget https://github.com/goharbor/harbor/releases/download/v2.8.1/harbor-offline-installer-v2.8.1.tgz
```
解压harbor离线安装包，重命名配置文件为harbor.yml
```shell
tar -zxf harbor-offline-installer-v2.8.1.tgz && cd harbor/ && mv harbor.yml.tmpl harbor.yml
```
修改配置文件
yaml文件严格要求缩进格式。注意`certificate:`和`private_key:`前面有两个空格
```shell
[root@ops harbor]# sed -i '/^hostname/s/^.*$/hostname: reg.myk8s.vm/' harbor.yml 
[root@ops harbor]# sed -i '/certificate:/s#^.*$#  certificate: /root/harbor_install/cert/reg.myk8s.vm.cert#' harbor.yml 
[root@ops harbor]# sed -i '/private_key:/s#^.*$#  private_key: /root/harbor_install/cert/reg.myk8s.vm.key#' harbor.yml
[root@ops harbor]# sed -i '/data_volume:/s#/data#/home/t4/harbor#' harbor.yml
```
修改hosts文件
```shell
ansible apps -m shell -a "sed -i '/app11/s/$/ reg.myk8s.vm/' /etc/hosts"
```
执行安装
等待提示安装完成
`docker ps -a`检查harbor容器是否全部`Up`
浏览器访问`https://192.168.122.11/`能够打开Harbor页面，使用默认账号密码`admin/Harbor12345`登入
```shell
./prepare && ./install.sh --with-notary
```
## k8s节点添加Harbor
k8s节点使用containerd作为容器运行时，则配置containerd对接Harbor
修改`/etc/containerd/config.toml`配置文件，在`      [plugins."io.containerd.grpc.v1.cri".registry.configs]`下添加Harbor信息
```shell
        [plugins."io.containerd.grpc.v1.cri".registry.configs."reg.myk8s.vm".tls]
          insecure_skip_verify = false
          ca_file = "/etc/containerd/cert/ca.crt"
          cert_file = "/etc/containerd/cert/reg.myk8s.vm.cert"
          key_file = "/etc/containerd/cert/reg.myk8s.vm.key"
        [plugins."io.containerd.grpc.v1.cri".registry.configs."reg.myk8s.vm.cn".auth]
          username = "admin"
          password = "Harbor12345"
```
配置containerd证书目录并放入证书文件
```shell
mkdir /etc/containerd/cert
cp /root/harbor_install/cert/ca.crt /etc/containerd/cert/
cp /root/harbor_install/cert/reg.myk8s.vm.cert /etc/containerd/cert/
cp /root/harbor_install/cert/reg.myk8s.vm.key /etc/containerd/cert/
```
分发修改好的配置文件和证书到k8s节点
```shell
for i in app1{2..3}; do scp /etc/containerd/config.toml $i:/etc/containerd/; scp -rp /etc/containerd/cert $i:/etc/containerd/cert; done
```
重启containerd服务
```shell
ansible apps -m shell -a "systemctl daemon-reload && systemctl restart containerd"
```
## ops机添加Harbor
ops机使用docker，则配置docker对接Harbor
配置docker证书目录并放入证书文件，然后重启docker服务
```shell
mkdir -p /etc/docker/certs.d/reg.myk8s.vm
cp /root/harbor_install/cert/ca.crt /etc/docker/certs.d/reg.myk8s.vm/
cp /root/harbor_install/cert/reg.myk8s.vm.cert /etc/docker/certs.d/reg.myk8s.vm/
cp /root/harbor_install/cert/reg.myk8s.vm.key /etc/docker/certs.d/reg.myk8s.vm/
systemctl restart docker
```
ops机重启docker服务会使部分Harobr容器退出，需要重新拉起Harbor
```shell
cd /root/harbor_install/harbor && docker compose up -d
```
## 验证ops机使用docker推送镜像到Harbor，k8s从Harbor拉取镜像部署pod
ops机从公网拉取一个测试镜像，改tag后上传到Harbor。使用默认账号密码`admin/Harbor12345`登入
```shell
docker pull busybox
docker login reg.myk8s.vm
docker tag busybox:latest reg.myk8s.vm/library/busybox:v1.0
docker push reg.myk8s.vm/library/busybox:v1.0
```
写一份pod.yaml测试
```shell
cat > pod.yaml <<-EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  containers:
  - name: test
    image: reg.myk8s.vm/library/busybox:v1.0
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c", "ping localhost -i 10"]
  restartPolicy: Never
EOF
```
创建pod并查看pod状态，`Running`表示k8s节点正常从Harbor拉取镜像
```shell
kubectl apply -f pod.yaml
kubectl get pod
```


# Kuboard
## 部署kuboard
将kuboard启动命令写入文件，方便维护。
避免和harbor端口冲突，web访问端口设置为8080。
```shell
[root@ops ~]# mkdir kuboard_install && mkdir /root/kuboard-data && cd kuboard_install
[root@ops kuboard_install]# cat > start-kuboard.sh <<-EOF
## 单机docker run方式运行kuboard
## 安装 Kuboard v3.x 版本的指令如下：
sudo docker run -d \
  --restart=unless-stopped \
  --name=kuboard \
  -p 8080:8080/tcp \
  -p 10081:10081/tcp \
  -e KUBOARD_ENDPOINT="http://192.168.122.11:8080" \
  -e KUBOARD_AGENT_SERVER_TCP_PORT="10081" \
  -v /root/kuboard-data:/data \
  eipwork/kuboard:v3
EOF
[root@ops kuboard_install]# bash start-kuboard.sh
```
## 访问Kuboard
在浏览器输入 `http://your-host-ip:8080` 即可访问 Kuboard v3.x 的界面，登录方式：
- 用户名： `admin`
- 默认密码： `Kuboard123`
