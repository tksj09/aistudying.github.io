# 使用kubespray部署Kubernetes

## 准备工作

### 主机规划   

| IP	| 作用 |
| ------ | ------ |
| 172.20.0.87	| ansible-client |
| 172.20.0.88	| master,node |
| 172.20.0.89	| master,node |
| 172.20.0.90	| node |
| 172.20.0.91	| node |
| 172.20.0.92	| node |


## 
### 关闭selinux
所有机器都必须关闭selinux，执行如下命令即可。
```
setenforce 0
sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```
### 网络配置
#### 在master机器上
```
firewall-cmd --permanent --add-port=6443/tcp
firewall-cmd --permanent --add-port=2379-2380/tcp
firewall-cmd --permanent --add-port=10250/tcp
firewall-cmd --permanent --add-port=10251/tcp
firewall-cmd --permanent --add-port=10252/tcp
firewall-cmd --permanent --add-port=10255/tcp
firewall-cmd --reload
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
sysctl -w net.ipv4.ip_forward=1
```
如果关闭了防火墙，则只需执行最下面三行。

#### 在node机器上
```
~]# firewall-cmd --permanent --add-port=10250/tcp
~]# firewall-cmd --permanent --add-port=10255/tcp
~]# firewall-cmd --permanent --add-port=30000-32767/tcp
~]# firewall-cmd --permanent --add-port=6783/tcp
~]# firewall-cmd  --reload
~]# echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
~]# sysctl -w net.ipv4.ip_forward=1
```
如果关闭了防火墙，则只需执行最下面两行。

#### 【可选】关闭防火墙
```
systemctl stop firewalld
```
## Kubespray部署k8s

### 在ansible-client机器上安装ansible

#### 安装ansible
```
sudo yum install epel-release
sudo yum install ansible
```
#### 安装jinja2
```
easy_install pip
pip2 install jinja2 --upgrade
```
如果执行pip2 install jinja2 --upgrade 出现类似如下的提示：
```
You are using pip version 9.0.1, however version 18.0 is available.
You should consider upgrading via the 'pip install --upgrade pip' command.
```
则执行``pip install --upgrade pip ``升级``pip``，再执行```pip2 install jinja2 --upgrade```

#### 安装Python 3.6
```
sudo yum install python36 –y
```
### 在ansible-client机器上配置免密登录
生成ssh公钥和私钥
在ansible-cilent机器上执行：

ssh-keygen
然后三次回车，生成ssh公钥和私钥。

建立ssh单向通道
在ansible-cilent机器上执行：
```
ssh-copy-id root@172.20.0.88		#将公钥分发给88机器
ssh-copy-id root@172.20.0.89
ssh-copy-id root@172.20.0.90
ssh-copy-id root@172.20.0.91
ssh-copy-id root@172.20.0.92
```
### 在ansible-client机器上安装kubespray
#### 下载kubespray

TIPS：本文下载的是master分支，如果大家要部署到线上环境，建议下载RELEASE分支。笔者撰写本文时，最新的RELEASE是2.6.0，RELEASE版本下载地址：https://github.com/kubernetes-incubator/kubespray/releases）

```
git clone https://github.com/kubernetes-incubator/kubespray.git
```
安装kubespray需要的包：
```
cd kubespray
sudo pip install -r requirements.txt
```
拷贝inventory/sample ，命名为inventory/mycluster ，mycluster可以改为其他你喜欢的名字
```
cp -r inventory/sample inventory/mycluster
```
使用inventory_builder，初始化inventory文件
```
declare -a IPS=(172.20.0.88 172.20.0.89 172.20.0.90 172.20.0.91 172.20.0.92)
CONFIG_FILE=inventory/mycluster/hosts.ini python36 contrib/inventory_builder/inventory.py ${IPS[@]}
```
此时，会看到inventory/mycluster/host.ini 文件内容类似如下：
```
[k8s-cluster:children]
kube-master      
kube-node        

[all]
node1    ansible_host=172.20.0.88 ip=172.20.0.88
node2    ansible_host=172.20.0.89 ip=172.20.0.89
node3    ansible_host=172.20.0.90 ip=172.20.0.90
node4    ansible_host=172.20.0.91 ip=172.20.0.91
node5    ansible_host=172.20.0.92 ip=172.20.0.92

[kube-master]
node1    
node2    

[kube-node]
node1    
node2    
node3    
node4    
node5    

[etcd]
node1    
node2    
node3    

[calico-rr]

[vault]
node1    
node2    
node3
```
### 使用ansible playbook部署kubespray
```
ansible-playbook -i inventory/mycluster/hosts.ini cluster.yml
```
大概20分钟左右，Kubernetes即可安装完毕。

验证
验证1：查看Node状态
```
kubectl get nodes
NAME      STATUS    ROLES         AGE       VERSION
node1     Ready     master,node   2m        v1.11.2
node2     Ready     master,node   2m        v1.11.2
node3     Ready     node          2m        v1.11.2
node4     Ready     node          2m        v1.11.2
node5     Ready     node          2m        v1.11.2
```
每个node都是ready的，说明OK。

验证2：部署一个NGINX


启动一个单节点nginx
```
kubectl run nginx --image=nginx:1.7.9 --port=80
```
为“nginx”服务暴露端口
```
kubectl expose deployment nginx --type=NodePort
```
查看nginx服务详情
```
kubectl get svc nginx
NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
nginx     NodePort   10.233.29.96   <none>        80:32345/TCP   14s
```
访问测试，如果能够正常返回NGINX首页，说明正常
```
curl localhost:32345
```
卸载
```
ansible-playbook -i inventory/mycluster/hosts.ini reset.yml
```


## 配置调整
接下来主要是调整一些配置。我们这里只安装核心组件，其它组件暂时先放弃，如果需要，大家可以按照下面的思路自己进行替换。

### 基础配置
这里主要是完成组件的选择。

在kubespray中有大量缺省配置，如果需要具体修改某一配置，可以按照如下思路进行查找。

例如我要修改etcd的运行内存分配大小。我们打开roles/etcd/defaults/main.yml，可以看到所有缺省配置。我们可以看到etcd_memory_limit这个配置。我们拷贝这个配置到inventory/test/group_vars/k8s-cluster.yml中，修改为我们想要的大小，这样在执行的时候，这个配置就会被复写。（不推荐直接修改role下的文件，以后更新的时候会很麻烦）

#### 组件部署模式
大部分组件部署模式都有3中，分别是Docker，host，和 rtk，由于我们不使用rtk的容器，所以不要把这些选项更改为rtk。如果你想保证主机的干净，我推荐绝大部分选择Docker部署就可以了。（kubelet排除在外，这个最好host，kubelet的容器化现在还在测试阶段）。我们这里挑选etcd配置和证书生成方案配置

#### etcd配置
打开：inventory/test/group_vars/k8s-cluster.yml，找到etcd_deployment_type这一部分，默认配置方式是就是以容器化启动etcd，我也推荐使用这个配置，如果有特殊需求，可以更改为host

#### 证书生成方案
打开：inventory/test/group_vars/all.yml，找到cert_management这一部分，默认配置方式是script，我也推荐使用这个配置，如果有特殊需求，可以更改为vault，至于什么是vault，可以百度一下。

这个配置主要是决定etcd，kubernetes的各种证书生成方式，默认使用脚本（openssl），至于vault，如果你曾经使用过，很了解，大力推荐。

#### 网络方案
Kubernetes 大部分安装完毕后，无法使用，都是因为网络问题。所以在这里我着重介绍一下网络。

#### 网段分配
首先我们需要确定的是pod和Service的网络，大家可以看一下关于Service的实现和CNI相关的部分。

选择网段逻辑：

无论是主机网络与容器网络，不能出现冲突。例如我们的办公网络是192.168.0.0/22，主机也在这个范围内。我的pod和Service的网络就不要随便的选择192.168.0.0/16的网络了（当然也可以选择192.168.0.0/16下的子网，只要不冲突就可以）。可以优先考虑从10.0.0.0/8或者172.16.0.0/12的子网进行选择。

网络段尽可能大，但是不要给的太过就可以。一般情况下pod网络，推荐使用x.x.x.x/16-18,Service 网络推荐使用x.x.x.x/18.就可以了。

具体计算：

办公网络192.168.0.0/22，pod和Service网络我选择在10.0.0.0/8的子网中。

一共最多有64个节点，每个节点最多254个容器。所以网络划分上

pod：10.233.0.0/18 （10.233.0.0->10.233.63.0） 0-63共64个节点

service： 10.233.64.0/18 （10.233.64.0->10.233.127.0） 最多有2^64 服务（分配过大，主要保证对称）。

#### 网络插件选择
kubespray* 支持很多网络插件，可以根据需要进行选择。

```
flannel: gre/vxlan (layer 2) networking.

calico: bgp (layer 3) networking.

canal: a composition of calico and flannel plugins.

cilium: layer 3/4 networking (as well as layer 7 to protect and secure application protocols), supports dynamic insertion of BPF bytecode into the Linux kernel to implement security services, networking and visibility logic.

contiv: supports vlan, vxlan, bgp and Cisco SDN networking. This plugin is able to apply firewall policies, segregate containers in multiple network and bridging pods onto physical networks.

weave: Weave is a lightweight container overlay network that doesn’t require an external K/V database cluster. (Please refer to weave troubleshooting documentation).
```

目前我推荐的calico，性能足够好，基于bgp实现。如果集群很小，可以考虑使用flannel。 如果想玩网络隔离，那么最好选择calico。

本次例子中，我们以calico为例，由于部署规模也很小，所以我们就不部署calico-rr。

修改配置
打开：inventory/test/group_vars/k8s-cluster.yml，修改如下配置：

```
kube_service_addresses: 10.233.0.0/18
kube_pods_subnet: 10.233.64.0/18
kube_network_plugin: calico
```



### 高可用方案
Kubernetes的高可用，要解决的核心其实是api组件的高可用和etcd的高可用，其它组件都还好。

#### etcd高可用
etcd本身就支持集群模式，所以啥都不用考虑，只要保证节点数量足够，升级备份之类的事情，kubespray已经帮你做了。

由于etcd采用Raft一致性算法，集群规模最好不要超过9个，推荐3，5，7，9个数量。具体看集群规模。如果性能不够，宁可多分配资源，也最好不要超过9个。

这里能选择偶数个节点吗？ 最好不要这样。原因有二：

偶数个节点集群不可用风险更高，表现在选主过程中，有较大概率或等额选票，从而触发下一轮选举。
偶数个节点集群在某些网络分割的场景下无法正常工作。试想，当网络分割发生后，将集群节点对半分割开。此时集群将无法工作。按照RAFT协议，此时集群写操作无法使得大多数节点同意，从而导致写失败，集群无法正常工作。

#### api 高可用

api的高可用，一般有2种思路。

各节点自己代理
loadbalancer_localhost

使用这种方式，会在每个Node节点启动一个Nginx代理，然后由这个Nginx代理服负载所有的master节点的api。master会访问自己节点下的api（localhost）。

外置负载均衡
利用外围的负载均衡实现，例如阿里云的SLB。自己搭建的就算了。完全不可靠啊，谨慎选择。

配置修改：

打开：``inventory/test/group_vars/all.yml``，修改如下配置：

```
# 负载均衡域名，提前配置好DNS解析

apiserver_loadbalancer_domain_name: "kubernetes.7mxing.com"

# 负载均衡

loadbalancer_apiserver:
  address: 10.0.128.160
  port: 6443
```

如果想要玩玩，可以参考官方的： https://github.com/kubernetes-incubator/kubespray/blob/master/docs/ha-mode.md 的haproxy实现

#### DNS方案

Kubernetes 的服务发现依赖DNS，其中又涉及2种类型的网络（host网络，容器网络），所以Kubespray 提供了2各配置用来管理。

dns_mode
主要用于集群内的DNS选择，推荐使用 dnsmasq_kubedns 或者全新的 coredns 。 coredns在之后会成为kubernetes 默认的DNS方案，目前还在测试。我们本次选择coredns

配置修改： 打开：inventory/test/group_vars/k8s-cluster.yml，修改如下配置：

dns_mode: coredns
resolvconf_mode
主要用来解决，当容器部署为host网络模式的时候，如何使用 kubernetes 的 DNS。 推荐使用 docker_dns，该配置会默认修改Docker的启动参数，设置Docker的DNS，非常方便。

配置修改：

打开：inventory/test/group_vars/k8s-cluster.yml，修改如下配置：

resolvconf_mode: docker_dns
其它组件
kubespray 还有很多组件可以选择，默认情况下的配置也足够绝大部分人使用了。这里我之所以介绍，主要是给大家留一些概念和具体使用方式。具体可以参考官方文档。

### 更换镜像

kubernetes的安装难倒绝大部分人的另外一个原因是防火墙，你懂的，你知道我在说什么。所以我们需要自己手动拉取镜像。

#### 获取镜像地址和版本

我们打开： roles/download/defaults/main.yml

我们最终选择的方案信息获取到我们需要的镜像和地址，（可能需要全局搜索，查找具体哪里使用了，如果你能看到）

kubernetes/coredns/calico

然后会得到这些地址信息：

```
# Etcd Images

etcd_image_repo: "quay.io/coreos/etcd"

# Hyperkube Images

hyperkube_image_repo: "gcr.io/google-containers/hyperkube"
pod_infra_image_repo: "gcr.io/google_containers/pause-amd64"

# calico Images

calicoctl_image_repo: "quay.io/calico/ctl"
calico_node_image_repo: "quay.io/calico/node"
calico_cni_image_repo: "quay.io/calico/cni"
calico_policy_image_repo: "quay.io/calico/kube-controllers"
calico_rr_image_repo: "quay.io/calico/routereflector"

# CoreDNS

coredns_image_repo: "docker.io/coredns/coredns"
```

版本信息：

```
# Versions

kube_version: v1.9.5
etcd_version: v3.2.4
calico_version: "v2.6.8"
calico_ctl_version: "v1.6.3"
calico_cni_version: "v1.11.4"
calico_policy_version: "v1.0.3"
calico_rr_version: "v0.4.2"
pod_infra_version: 3.0
```

#### 拉取镜像

这些镜像在国外，就需要借助我们的VPS或者代理了。拉下来这些镜像后，我们需要一个docker registry，这里我们选择阿里云。

至于如何使用VPS，或者如何给Docker配置代理，请自行百度了。

阿里云
在你的VPS或者本地安装好Docker，然后在阿里云上建立一个分组。使用docker login进行登陆。这样就配置完毕了。

例如我们在最终确认我们的分组地址为： registry.cn-beijing.aliyuncs.com/bbt-kube

创建拉取脚本
根据上面的的镜像地址和版本信息，我们创建一个简单的脚本用来拉取所需要的所有镜像。脚本如下：

```shell
# !/usr/bin/env bash

images1=(
 	etcd:v3.2.4
)

images2=(
 	ctl:v1.6.3
 	node:v2.6.8
 	cni:v1.11.4
 	kube-controllers:v1.0.3
 	routereflector:v0.4.2
)

images3=(
 	hyperkube:v1.9.5
 	pause-amd64:3.0
)

images4=(
 	coredns:1.1.0
)

for imageName in ${images1[@]} ; do
     docker pull quay.io/coreos/$imageName
     docker tag quay.io/coreos/$imageName registry.cn-beijing.aliyuncs.com/bbt-kube/$imageName
     docker push registry.cn-beijing.aliyuncs.com/bbt-kube/$imageName
done

for imageName in ${images2[@]} ; do
     docker pull quay.io/calico/$imageName
     docker tag quay.io/calico/$imageName registry.cn-beijing.aliyuncs.com/bbt-kube/$imageName
     docker push registry.cn-beijing.aliyuncs.com/bbt-kube/$imageName
done

for imageName in ${images3[@]} ; do
     docker pull gcr.io/google_containers/$imageName
     docker tag gcr.io/google_containers/$imageName registry.cn-beijing.aliyuncs.com/bbt-kube/$imageName
     docker push registry.cn-beijing.aliyuncs.com/bbt-kube/$imageName
done

for imageName in ${images4[@]} ; do
     docker pull docker.io/coredns/$imageName
     docker tag docker.io/coredns/$imageName registry.cn-beijing.aliyuncs.com/bbt-kube/$imageName
     docker push registry.cn-beijing.aliyuncs.com/bbt-kube/$imageName
done
```

然后chomd +x all.sh && ./all.sh，执行完毕后，理论上，所有镜像都会被推送到阿里云的镜像仓库中。回到阿里云的镜像仓库管理下，我们将所有镜像设置为公有

docker_registry

这样我们就完成了我们所有镜像的拉取了。

如果你没有VPS或者代理，可以直接跳过这一步（前提是组件选择的和我的差不多）。

#### 更换镜像地址

接下来我们就要替换镜像拉取地址。

打开：** inventory/test/group_vars/k8s-cluster.yml**， 追加如下内容：

```
# Docker Pull Url

docker_images_pull_url: "registry.cn-beijing.aliyuncs.com/bbt-kube"

# Etcd Images

etcd_image_repo: "/etcd"

# Hyperkube Images

hyperkube_image_repo: "/hyperkube"
pod_infra_image_repo: "/pause-amd64"

# calico Images

calicoctl_image_repo: "/ctl"
calico_node_image_repo: "/node"
calico_cni_image_repo: "/cni"
calico_policy_image_repo: "/kube-controllers"
calico_rr_image_repo: "/routereflector"

# CoreDNS

coredns_image_repo: "/coredns"
```

这样所有组件的配置都会用我们的镜像地址来替换了。

### 其它调优

前面已经完成了绝大部分了，下面主要是一些调优。

Docker安装源更改
默认从docker官方源安装docker，速度太慢了，我们更换为国内源。

在 inventory/test/group_vars/k8s-cluster.yml， 追加如下内容：

```
# Debian docker-ce repo

docker_debian_repo_base_url: "https://mirrors.aliyun.com/docker-ce/linux/debian"
docker_debian_repo_gpgkey: 'https://mirrors.aliyun.com/docker-ce/linux/debian/gpg'
dockerproject_apt_repo_base_url: 'https://mirrors.aliyun.com/docker-engine/apt/repo'
dockerproject_apt_repo_gpgkey: 'https://mirrors.aliyun.com/docker-engine/apt/gpg'
```

具体配置可以参考： role/docker/defaults/main.yml

Docker mirrors
为了加快镜像拉取速度，我们配置了docker mirrors。

在 inventory/test/group_vars/k8s-cluster.yml， 修改如下内容：

docker_options: "--insecure-registry= --registry-mirror=https://iusfdkrx.mirror.aliyuncs.com --graph=  "
具体配置可以参考： role/docker/defaults/main.yml

扩展组件安装
由于我们使用NFS做存储，所有我需要保证所有主机上都安装了nfs-common

在 inventory/test/group_vars/k8s-cluster.yml， 追加如下内容：

  ```
  common_required_pkgs:
  
  - python-httplib2
  - openssl
  - curl
  - rsync
  - bash-completion
  - socat
  - unzip
  - nfs-common
  ```

  具体配置可以参考： role/kubernetes/preinstall/defaults/main.yml

OK 这样的准备工作我们已经完成。接下来就愉快的安装吧。

使用
进入kubespray的目录，接下来愉快的玩把。

安装
```
ansible-playbook -i inventory/test/hosts.ini cluster.yml --private-key=inventory/test/test.pem
```
环境清理
```
ansible-playbook -i inventory/test/hosts.ini reset.yml --private-key=inventory/test/test.pem
```
其它
可以参考官方文档。

最终结果
拷贝第一个maste的节点下的/root/.kube/config到本地的~/.kube/config。
