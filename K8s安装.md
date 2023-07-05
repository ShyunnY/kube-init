

# K8s1.27搭建(containerd)



## 0. 环境准备

本次K8s环境搭建基于vmware,系统版本为CentOS7.9。

| 主机名 |       IP        | 配置 |
| :----- | :-------------: | ---- |
| Master | 192.168.136.129 | 2c2g |
| node1  | 192.168.136.130 | 1c2g |
| node2  | 192.168.136.131 | 1c2g |

> 本文所涉及到的配置文件都上传至仓库，各位可以拉下来使用,有些需要修改的，按照本文进行修改即可。
>
> URL：



## 1. 安装CRI(containerd)

自 1.24 版起，Dockershim 已从 Kubernetes 项目中移除。

所以本文将使用containerd作为K8s的CRI

**配置转发 IPv4 并让 iptables 看到桥接流量**

```shell
$ cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

$ modprobe overlay
$ modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
$ sysctl --system

# 通过运行以下指令确认 br_netfilter 和 overlay 模块被加载：
lsmod | grep br_netfilter
lsmod | grep overlay

# 通过运行以下指令确认 net.bridge.bridge-nf-call-iptables、net.bridge.bridge-nf-call-ip6tables 和 net.ipv4.ip_forward 系统变量在你的 sysctl 配置中被设置为 1：
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```



### 1.1安装containerd

```shell
# 下载containerd
$ wget https://github.com/containerd/containerd/releases/download/v1.7.2/containerd-1.7.2-linux-amd64.tar.gz

# 解压
$ tar Cxzvf /usr/local containerd-1.6.2-linux-amd64.tar.gz
bin/
bin/containerd-shim-runc-v2
bin/containerd-shim
bin/ctr
bin/containerd-shim-runc-v1
bin/containerd
bin/containerd-stress

# 下载 systemd文件,让containerd可以systemd运行
$ wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

# 移动到 `/usr/local/lib/systemd/system`下
$ mkdir /usr/local/lib/systemd/system -p
$ mv containerd.service /usr/local/lib/systemd/system/containerd.service

# 开启containerd
$ systemctl daemon-reload
$ systemctl enable --now containerd
```



### 1.2安装runc

```shell
# 下载runc
$ wget https://github.com/opencontainers/runc/releases/download/v1.1.7/runc.amd64

# 安装runc
$ install -m 755 runc.amd64 /usr/local/sbin/runc
```



### 1.3安装CNI插件

```shell
# 下载CNI插件
$ wget https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz

# 新建目录并进行解压
$ mkdir -p /opt/cni/bin
$ tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
./
./macvlan
./static
./vlan
./portmap
./host-local
./vrf
./bridge
./tuning
./firewall
./host-device
./sbr
./loopback
./dhcp
./ptp
./ipvlan
./bandwidth
```



### 1.4配置containerd Config

首先我们先生成`containerd`的默认模板

```shell
# 生成默认文件 
$ containerd config default | sudo tee /etc/containerd/config.toml
```

在默认文件中我们有几个点需要改。

**首先是结合 `runc` 使用 `systemd` cgroup 驱动。**

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
			......
			# 开启SystemdCgroup
            SystemdCgroup = true
```

**其次修改sandbox沙箱镜像地址**

```toml
[plugins."io.containerd.grpc.v1.cri"]
	......
	# 修改为阿里云的镜像地址
    sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.8"
```

**最后再设置镜像加速器**

```toml
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://阿里云镜像加速器地址"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["https://gcr.k8s.li"]
```

> 详细配置文件在github仓库中，使用前请检查上述配置。

**我们重启containerd,更新一下配置**

```shell
$ systemctl daemon-reload
$ systemctl enable --now containerd

# 查看一下版本
$ ctr version
```





## 2.搭建集群前的环境准备

> ⭐以下操作各个节点都必须配置！！！

设置/etc/hosts

```shell
192.168.136.129 master
192.168.136.130 node-1
192.168.136.131 node-2
```

> 每个节点都需要配置。

关闭防火墙

```shell
 systemctl stop firewalld && systemctl disable firewalld
 systemctl stop NetworkManager && systemctl disable NetworkManager
```

关闭swap交换

```shell
 swapoff -a
 sed -ri 's/.*swap.*/#&/' /etc/fstab
```

配置时间同步

```shell
yum -y install ntpdate
ntpdate ntp1.aliyun.com
```

SELinux禁用

```shell
 setenforce 0
 sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

yum源配置**(用于下载kubeadm,kubelet,kubectl)**

```shell
 cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
        http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

下载K8s组件
```shell
 yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

开启kubelet自启动

```shell
 systemctl enable kubelet.service --now
```



## 3.搭建K8s集群

#### Mater

首先生成默认的配置文件

```shell
# 生成默认配置文件
$ kubeadm config print init-defaults > kubeadm.yml
```

注意,在默认配置文件中我们有几个需要修改的地方。

+ **advertiseAddress:** 填写master地址
+ **imageRepository:** 修改为国内镜像源
+ **nodeRegistration.criSocket:** 1.24后默认为containerd
+ **nodeRegistration.name:** 修改为节点名，即master

> 完整文件放在**github**上,请对照上述进行修改

查看需要的镜像,先手动拉取一次(<u>头铁的可以跳过这步直接初始化</u>)

```shell
# 查看所需的镜像
$ kubeadm config images list --config kubeadm.yml
registry.aliyuncs.com/google_containers/kube-apiserver:v1.27.0
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.27.0
registry.aliyuncs.com/google_containers/kube-scheduler:v1.27.0
registry.aliyuncs.com/google_containers/kube-proxy:v1.27.0
registry.aliyuncs.com/google_containers/pause:3.9
registry.aliyuncs.com/google_containers/etcd:3.5.7-0
registry.aliyuncs.com/google_containers/coredns:v1.10.1

# 手动拉取
$ for img in `kubeadm config images list --config kubeadm.yml` ; do ctr i pull $img ; done
```

接下来就可以开始初始化主节点了

```shell
# --upload-certs 
$ kubeadm init --config=kubeadm.yml --upload-certs | tee kubeadm.log
```

当你看到这个,就代表成功了🎉🎉🎉

```shell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

# node节点join直接使用
kubeadm join 192.168.136.129:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:7693b522f5a173f7bc779e0b6c52e77dc59fd416630f1d0de7dac0e7a2c3ce82
```

按照指示,执行命令

```shell
# kubeconfig配置 
# 后续kubectl等组件将使用该config
# 如果不设置 kubectl无法正常使用
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 环境变量设置
export KUBECONFIG=/etc/kubernetes/admin.conf
```



#### Node

搭建node就简单多了，直接将上面的命令拷过来即可。**(需要在内网可以相互ping通下执行！！！)**

在node1,node2上执行

```shell
# token为1h过期，如果过期了重新申请即可
kubeadm join 192.168.136.129:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:7693b522f5a173f7bc779e0b6c52e77dc59fd416630f1d0de7dac0e7a2c3ce82
```

在master上我们可以查看一下节点状态。

因为CNI还没进行安装，故`STATUS`为NotReady。

接下来我们继续安装**Calico**

```shell
$ kubectl get node 
NAME     STATUS     ROLES           AGE     VERSION
master   NotReady   control-plane   5m48s   v1.27.3
node-1   NotReady   <none>          111s    v1.27.3
node-2   NotReady   <none>          107s    v1.27.3
```

## 4.安装Calico

首先我们在**node1,node2**中下载`Calico`的manifest清单。(*由于master污点存在,工作负载不会调度到master上*)

```shell
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml -O
```

> 这里我用的是最新版，如果搭建旧版K8s，一定要注意calico和K8s的版本关系。

亲测,通过手动拉取镜像的方式会稍微快一点**(虽然也很慢)**,避免`kubectl apply -f file`超时。

```shell
# 找到calico.yaml中CLUSTER_TYPE 
# 在其下面添加以下配置
- name: IP_AUTODETECTION_METHOD
  value: "interface=ens.*"
  
# 手动拉取image (这里会异常的慢 耐心等待)
for img in `cat calico.yml |grep docker.io|awk {'print $2'}` ; do ctr i pull $img ; done
```

拉取完成后,我们在master节点上部署calico

```shell
# 部署calico
$ kubectl apply -f calico.yaml

# 稍微等待
# 查看calico是否启动完成
$ kubectl get pod -n kube-system
```



## 5. 测试

至此,我们将集群基本搭建起来。我们通过部署**nginxPod**测试功能是否正常。

配置文件:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "test"
  namespace: default
spec:
  containers:
  - name: test
    image: "nginx:1.14.1"
    ports:
    - containerPort:  80
  restartPolicy: Always
```

等待镜像拉取后部署即可。

```shell
# 查看pod
$ kubectl get pod -o wide
NAME   READY   STATUS    RESTARTS   AGE   IP              NODE     NOMINATED NODE   READINESS GATES
test   1/1     Running   0          8s    172.16.84.139   node-1   <none>           <none>

# 测试请求
$ curl 172.16.84.139
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

至此基于containerd的K8s1.27版本搭建完成。



## 6.疑难杂症



### 1.使用wget拉取containerd速度过慢

我的建议是本地下载containerd,runc,cni后传到服务器上。



### 2.kubeadm init报错Initial timeout of 40s passed.

在主节点上先把镜像都拉下来，然后再执行`kubeadm init`操作



### 3.在节点上执行crictl ps报错

执行以下命令

```shell
$ crictl config --set runtime-endpoint=unix:///run/containerd/containerd.sock --set image-endpoint=unix:///run/containerd/containerd.sock
```

此时执行`crictl ps`可以看到节点上运行的pod

```shell
# 查看正在运行的 container
$ crictl ps

CONTAINER           IMAGE               CREATED             STATE               NAME                      ATTEMPT             POD ID              POD
94656931edbf3       ead0a4a53df89       5 hours ago         Running             coredns                   1                   769d609e95634       coredns-7bdc4cb885-9w7lg
c90acc2377df5       ead0a4a53df89       5 hours ago         Running             coredns                   1                   3fdaa6c6d5058       coredns-7bdc4cb885-vlhzc
4a449e3f47d1c       1919f2787fa70       5 hours ago         Running             calico-kube-controllers   1                   880f6d2cce51e       calico-kube-controllers-85578c44bf-f4bgn
ef95d543787bf       8065b798a4d67       5 hours ago         Running             calico-node               1                   1d50c3b758d40       calico-node-ffnwv
30da8dfeb2fcf       5f82fc39fa816       5 hours ago         Running             kube-proxy                1                   8924433fcc6f3       kube-proxy-kssm4
dc79bfc62a93d       f73f1b39c3fe8       5 hours ago         Running             kube-scheduler            1                   c65802fdadf37       kube-scheduler-master
28cdd1bbb9ff3       86b6af7dd652c       5 hours ago         Running             etcd                      1                   7846dfcf4734e       etcd-master
34ef37f8da317       6f707f569b572       5 hours ago         Running             kube-apiserver            1                   34a77a69c3de5       kube-apiserver-master
5a862db1b835e       95fe52ed44570       5 hours ago         Running             kube-controller-manager   1                   2f9ebde24aa43       kube-controller-manager-master
```

