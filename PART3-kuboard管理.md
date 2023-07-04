# Kuboard管理
Kuboard官方在线教程：`https://kuboard.cn/learning/`

## 添加集群
按页面引导添加集群
需要注意的是使用导入.kubeconfig方式添加集群的话，可能发生报错：`【Kuboard 不能连接 APIServer】 Post "https://192.168.122.12:6443/apis/authorization.k8s.io/v1/selfsubjectrulesreviews": x509: certificate signed by unknown authority【您可以尝试在此处使用 Master 节点的真实 IP】`
尝试以下解决方法：
1. 如果kuboard版本是v3.3.0.0或更低，则升级到v3.5.2.4
2. 如果kuboard版本已经升级到v3.5.2.4，粘贴.kubeconfig后不要去选择context下拉框里的选项，保持默认值直接点“确定”

## 添加nfs类型存储类
为ops虚拟机添加一块硬盘sdb，创建主分区sdb1，格式化为xfs格式，挂载到/data目录
```shell
mkdir /data
mkfs.xfs /dev/sdb1
echo '/dev/sdb1 /data xfs defaults 0 0' >> /etc/fstab
mount -a
```
给所有机器安装nfs-utils
```shell
ansible apps -m shell -a 'yum -y install nfs-utils'
```
ops配置/data作为nfs共享目录
```shell
echo '/data 192.168.122.*(insecure,rw,sync,no_root_squash)' >> /etc/exports
systemctl status nfs-server.service
```
在kuboard页面创建nfs类型存储类
![image](https://github.com/fiendxiu/myk8s_paas/assets/24711307/4c37d232-b3f5-48a4-b8a5-d136ec71ff7c)

##
