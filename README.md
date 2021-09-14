# -rancher-RKE

Rancher 中文文档
https://docs.rancher.cn/

4C 8G Centos7.7  3 master+2 node

文件下载地址
http://mirror.cnrancher.com/

1. 安装RKE工具
https://github.com/rancher/rke/releases 
master节点01部署
1.1 下载RKE工具到本地
 wget http://rancher-mirror.cnrancher.com/rke/v1.1.1/rke_linux-amd64
chmod +x rke_linux-amd64 && sudo  mv rke_linux-amd64 /usr/bin/rke

1.2 查看当前RKE版本
rke --version

1.3 查看RKE支持的Kubernetes版本
rke config --list-version --all

1.4 查看RKE支持的image
rke config --system-images --all

2. Docker环境准备
2.1 安装Docker
#定义用户名
NEW_USER=rancher
#添加用户(可选)
sudo adduser $NEW_USER
#为新用户设置密码
echo rancher | sudo passwd $NEW_USER --stdin
#为新用户添加sudo权限
sudo echo "$NEW_USER ALL=(ALL) ALL" >> /etc/sudoers
#卸载旧版本Docker软件
sudo yum remove docker \
docker-client \
docker-client-latest \
docker-common \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-selinux \
docker-engine-selinux \
docker-engine \
container*

#安装必要的一些系统工具
sudo yum install -y yum-utils device-mapper-persistent-data  lvm2 bash-completion
#添加Docker源信息
sudo yum-config-manager --add-repo \     http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#查看docker的版本
yum list docker-ce --showduplicates | sort -r
# 安装docker 19.03.7版本
sudo yum -y install docker-ce-19.03.7-3.el7 docker-ce-cli-19.03.7-3.el7 containerd.io
#把当前用户加入docker组
sudo usermod -aG docker $NEW_USER
#设置开机自启并运行docker服务
sudo systemctl enable --now docker

2.2 锁定Docker版本
# 安装yum-plugin-versionlock插件
yum -y install yum-plugin-versionlock

# 锁定Docker软件包
yum versionlock add docker-ce-19.03.7-3.el7 docker-ce-cli-19.03.7-3.el7 containerd.io

# 查看已锁定的软件包
yum versionlock list

# 解锁指定软件包
yum versionlock delete <软件包名称>

#解锁所有软件包
yum versionlock clear

3. 系统内核调优
cat >> /etc/sysctl.d/kubernetes.conf<<EOF
# 开启路由功能
net.ipv4.ip_forward=1
# 避免cpu资源长期使用率过高导致系统内核锁
kernel.watchdog_thresh=30
# 开启iptables bridge
net.bridge.bridge-nf-call-iptables=1
# 调优ARP高速缓存
net.ipv4.neigh.default.gc_thresh1=4096
net.ipv4.neigh.default.gc_thresh2=6144
net.ipv4.neigh.default.gc_thresh3=8192
EOF
sysctl -p && systemctl restart docker

docker服务所有节点都部署
docker安装脚本

#! /bin/bash
#安装Docker
#定义用户名
NEW_USER=rancher
#添加用户(可选)
sudo adduser $NEW_USER
#为新用户设置密码
echo rancher | sudo passwd $NEW_USER --stdin
#为新用户添加sudo权限
sudo echo "$NEW_USER ALL=(ALL) ALL" >> /etc/sudoers
#安装必要的一些系统工具
sudo yum install vim wget bash-completion lrzsz nmap nc tree htop iftop net-tools -y
sudo yum install -y yum-utils device-mapper-persistent-data  lvm2 bash-completion
#添加Docker源信息
sudo yum-config-manager --add-repo \
    http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#缓存docker源
sudo yum makecache
#安装docker 19.03.7版本
sudo yum -y install docker-ce-19.03.7-3.el7 docker-ce-cli-19.03.7-3.el7 containerd.io
#把当前用户加入docker组
sudo usermod -aG docker $NEW_USER
#设置开机自启并运行docker服务
sudo systemctl enable --now docker
#安装yum-plugin-versionlock插件
yum -y install yum-plugin-versionlock
#锁定Docker软件包
yum versionlock add docker-ce-19.03.7-3.el7 docker-ce-cli-19.03.7-3.el7 containerd.io
#关闭虚拟内存
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab
#系统内核调优
cat >> /etc/sysctl.d/kubernetes.conf<<EOF
# 开启路由功能
net.ipv4.ip_forward=1
# 避免cpu资源长期使用率过高导致系统内核锁
kernel.watchdog_thresh=30
# 开启iptables bridge
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables = 1
# 调优ARP高速缓存
net.ipv4.neigh.default.gc_thresh1=4096
net.ipv4.neigh.default.gc_thresh2=6144
net.ipv4.neigh.default.gc_thresh3=8192
EOF
sysctl -p && systemctl restart docker
#配置加速器

sudo tee /etc/docker/daemon.json <<-'EOF'
{
"max-concurrent-downloads": 3,
"max-concurrent-uploads": 5,
"registry-mirrors": ["https://0bb06s1q.mirror.aliyuncs.com"],
"storage-driver": "overlay2",
"storage-opts": ["overlay2.override_kernel_check=true"],
"log-driver": "json-file",
"log-opts": {
  "max-size": "100m",
  "max-file": "3"
}
}
EOF
#重启docker
systemctl daemon-reload && systemctl restart docker && systemctl enable docker.service
#查看docker版本信息
sudo docker info



4. 同步/etc/hosts
# Kubernetes cluster demo1
192.168.31.130  rmaster01
192.168.31.131  rmaster02
192.168.31.132  rmaster03
192.168.31.133  node01
192.168.31.134  node02

master01节点操作
5. 配置rancher用户ssh单向无密码访问
# 所有节点执行,注意首先切换为rancher
su - rancher
ssh-keygen -t rsa

# 在rmaster01配置ssh单向无密码访问
ssh-copy-id rmaster01
ssh-copy-id rmaster02
ssh-copy-id rmaster03
ssh-copy-id node01
ssh-copy-id node02

# 测试
for i in `cat /etc/hosts | grep -v localhost | grep -Ev '^$|#' | awk '{print $2}'`;do ssh $i hostname;done

阿里云配置内网地址

5.1.生成cluster.yml配置文件
cat << EOF >  cluster.yml
nodes:
  - address: 172.31.53.130
    hostname_override: rmaster01
    internal_address:
    user: rancher
    role: [controlplane,etcd]
  - address: 172.31.53.131
    hostname_override: rmaster02
    internal_address:
    user: rancher
    role: [controlplane,etcd]
  - address: 172.31.53.132
    hostname_override: rmaster03
    internal_address:
    user: rancher
    role: [controlplane,etcd]
  - address: 172.31.53.133
    hostname_override: node01
    internal_address:
    user: rancher
    role: [worker]
  - address: 172.31.53.134
    hostname_override: node02
    internal_address:
    user: rancher
    role: [worker]

# 定义kubernetes版本
kubernetes_version: v1.17.5-rancher1-1

# 如果要使用私有仓库中的镜像，配置以下参数来指定默认私有仓库地址。
#private_registries:
#    - url: registry.com
#      user: Username
#      password: password
#      is_default: true

services:
  etcd:
    # 扩展参数
    extra_args:
      # 240个小时后自动清理磁盘碎片,通过auto-compaction-retention对历史数据压缩后，后端数据库可能会出现内部碎片。内部碎片是指空闲状态的，能被后端使用但是仍然消耗存储空间，碎片整理过程将此存储空间释放回文>件系统
      auto-compaction-retention: 240 #(单位小时)
      # 修改空间配额为6442450944，默认2G,最大8G
      quota-backend-bytes: '6442450944'
    # 自动备份
    snapshot: true
    creation: 5m0s
    retention: 24h
  kubelet:
    extra_args:
      # 支持静态Pod。在主机/etc/kubernetes/目录下创建manifest目录，Pod YAML文件放在/etc/kubernetes/manifest/目录下
      pod-manifest-path: "/etc/kubernetes/manifest/"
# 有几个网络插件可以选择：flannel、canal、calico，Rancher2默认canal
network:
  plugin: canal
  options:
    flannel_backend_type: "vxlan"
# 可以设置provider: none来禁用ingress controller
ingress:
  provider: nginx
  node_selector:
    app: ingress
EOF

查看RKE支持的Kubernetes版本
rke config --list-version --all
[rancher@rmaster01 ~]$ rke config -l --all
v1.17.17-rancher2-1
v1.16.15-rancher1-4
v1.18.16-rancher1-1
v1.19.8-rancher1-1
v1.20.4-rancher1-1
[rancher@rmaster01 ~]$
5.2. 部署kubernetes集群
rke up --config ./cluster.yml
