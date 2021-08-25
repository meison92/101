## 参考
[使用kubeadmin安装部署Kubernetes](http://www.qishunwang.net/news_show_11603.aspx)

### 设置防火墙
```
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
sysctl --system
```

### 关闭swap、关闭防火墙、禁用Selinux
```
#关闭磁盘交换
临时关闭
sudo swapoff -a
永久关闭
sed -i 's/.*swap.*/#&/' /etc/fstab

防火墙
ufw status
ufw disable
或者：
# 关闭防火墙
systemctl stop firewalld
# 关闭防火墙开机启动
systemctl disable firewalld

### 禁用Selinux
# 临时关闭
sudo apt install selinux-utils
setenforce 0
# 永久关闭
sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```


### why:
不止部署k8s，许多公司在装机过程就就直接关闭了swap、selinux和防火墙
selinux，这个是用来加强安全性的一个组件，但非常容易出错且难以定位，一般上来装完系统就先给禁用了

iptables防火墙，会对所有网络流量进行过滤、转发，如果是内网机器一般都会直接关闭，省的影响网络性能，但k8s不能直接关了，k8s是需要用防火墙做ip转发和修改的，当然也看使用的网络模式，如果采用的网络模式不需要防火墙也是可以直接关闭的

swap，这个当内存不足时，linux会自动使用swap，将部分内存数据存放到磁盘中，这个这样会使性能下降，为了性能考虑推荐关掉

官方解释：部署文档上都有说明原因。
关于防火墙的原因：
（nftables后端兼容性问题，产生重复的防火墙规则）Theiptablestooling can act as a compatibility layer, behaving like iptables but actually configuring nftables. This nftables backend is not compatible with the current kubeadm packages: it causes duplicated firewall rules and breakskube-proxy.

关于selinux的原因：
（关闭selinux以允许容器访问宿主机的文件系统）Setting SELinux in permissive mode by runningsetenforce 0andsed ...effectively disables it. This is required to allow containers to access the host filesystem, which is needed by pod networks for example. You have to do this until SELinux support is improved in the kubelet.

至于swap：
[github.com/kubernetes/](https://link.zhihu.com/?target=https%3A//github.com/kubernetes/kubernetes/issues/53533)



## install
[Ubuntu下安装kubernetes实践](https://blog.csdn.net/pzyyyyy/article/details/104396710)  
[Kubernetes 基于 ubuntu18.04 手工部署 (k8s)](https://www.cnblogs.com/xiaoxuebiye/p/11256292.html)  
[我的k8s随笔：Kubernetes 1.17.0 部署讲解](https://latelee.blog.csdn.net/article/details/103774072)  
[Ｂ站部署视频](https://www.bilibili.com/video/BV1tJ411N7Zx?from=search&seid=12454234905975326572)

```vim
/etc/apt/sources.list 添加
deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
或
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
EOF

执行
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt update

执行
sudo apt install -y kubelet kubeadm kubectl

阻止自动更新(apt upgrade时忽略)
sudo apt-mark hold kubelet kubeadm kubectl
```

#### coredns问题
```bash
docker pull registry.aliyuncs.com/google_containers/coredns:<version>
docker tag registry.aliyuncs.com/google_containers/coredns:<version> k8s.gcr.io/coredns:v<version>
docker rmi registry.aliyuncs.com/google_containers/coredns:<version>
```

#### 获取部署所需的镜像版本
```bash
kubeadm config images list
kubeadm config images list --kubernetes-version=v1.22.1
拉取镜像
kubeadm config images pull
```

## start
```bash
bashkubectl version --client 查看版本

kubeadm init --kubernetes-version=v1.22.1 \
 --apiserver-advertise-address=10.64.4.50 \
 --image-repository registry.aliyuncs.com/google_containers \
 --service-cidr=10.96.0.0/12
 --pod-network-cidr=10.244.0.0/16 \
 --ignore-preflight-errors=all --v=6

 kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

| Syntax      | Description |
| ----------- | ----------- |
|--kubernetes-version=v1.22.1| 这个参数是下载的k8s软件版本号 |
| --apiserver-advertise-address=10.64.4.50 | 这个参数就是master主机的IP地址，例如我的Master主机的IP是：10.64.4.50 |
| --image-repository=registry.aliyuncs.com/google_containers | 这个是镜像地址，由于国外地址无法访问，故使用的阿里云仓库地址：registry.aliyuncs.com/google_containers |
| --service-cidr=10.96.0.0/12 | 用于指定为Service分配使用的网络地址，它由kubernetes管理，默认即为10.96.0.0/12 |
| --pod-network-cidr=10.244.0.0/16 | 用于指定分Pod分配使用的网络地址，它通常应该与要部署使用的网络插件（例如flannel、calico等）的默认设定保持一致，10.244.0.0/16是flannel默认使用的网络.<br> flannel定义在　https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml |


## 重置kubeadm
```bash
kubeadm reset
```
## Post run
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf
```

## untaint master
```bash
kubectl taint nodes --all node-role.kubernetes.io/master-
```

### 新建一个nginx的deployment
```vim nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx1
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx2
  template:
    metadata:
      labels:
        app: nginx2
    spec:
      containers:
        - name: nginx3
          image: nginx


使用：
kubectl create -f httpserver.yaml
kubectl get po -owide
curl <nginx-ip>

之后可能要配一个service，来做一个负载均衡．
```


```vim httpserver.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpserver
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: httpserver
    spec:
      containers:
      -name: cncamp/httpserver:v1.0
       image: httpserver


使用：
git clone https://github.com/cncamp/golang
cd golang/httpserver/ && make push
kubectl create -f httpserver.yaml
kubectl get po -owide　获取httpserver的ip地址．

curl <httpserver-ip>

```
最简单的nginx pod:
```vim　pod_nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80

使用：
kubectl create -f　pod_nginx.yaml
```


## 配置节点
主节点主机运行以下命令，得到次节点的加入命令
```bash
kubeadm token create --print-join-command
```
次节点运行
```
 kubeadm join 10.64.4.50:6443 --token nfka8w.17au4y3uny5qosti --discovery-token-ca-cert-hash sha256:c70fbf25f2d19035c311e1d1f75cad1ee2ad0bae3c2c21debf7a11ad39f7295a
```
在主节点上获取所有pods，并按照node排序
```bash
kubectl get pods -A -o wide --sort-by="{.spec.nodeName}"
```

## 安装dashboard

[K8s安装dashboard](https://www.cnblogs.com/bigberg/p/13469736.html)
```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.1/aio/deploy/recommended.yaml
```


## stop k8s
[kubernetes优雅停止](https://segmentfault.com/a/1190000023774462)
```bash
关机过程
0.node节点
systemctl stop kube-proxy
systemctl stop kubelet

1.master
systemctl stop kube-scheduler
systemctl stop kube-controller-manager
systemctl stop kube-apiserver

2.关闭 node节点的flanneld 服务
systemctl stop flanneld

3.全部节点关闭etcd
systemctl stop etcd
systemctl stop docker

4.全部关机
init 0

 貌似调用 kubeadm reset 也可以reset

开机过程
systemctl start etcd  三节点
systemctl start flanneld （node 节点）
systemctl start kube-apiserver  (默认设置了开机启动， master 节点)
systemctl start kube-scheduler （默认设置了开机启动， master 节点）
systemctl start kube-controller-manager （默认设置了开机启动， master 节点）
systemctl start kubelet（node 节点）
systemctl start kube-proxy（node 节点）
```


### 安装calico
```bash
kubectl apply -f https://docs.projectcalico.org/v3.6/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
kubectl apply -f https://git.io/weave-kube-1.6
```

### 问题：
在安装kubernetes的过程中，会出现
```bash
failed to create kubelet: misconfiguration: kubelet cgroup driver: "cgroupfs" is different from docker cgroup driver: "systemd"
```
文件驱动默认由systemd改成cgroupfs, 而我们安装的docker使用的文件驱动是systemd, 造成不一致, 导致镜像无法启动

docker info查看
Cgroup Driver: `systemd`

现在有两种方式, 一种是修改docker, 另一种是修改kubelet,

修改docker:#

修改或创建/etc/docker/daemon.json，加入下面的内容：
```vim
{

  "exec-opts": ["native.cgroupdriver=systemd"]

}
```

重启docker:
```bash
systemctl restart docker
systemctl status docker
```
修改kubelet:#

vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```bash
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
```

添加如下内容--cgroup-driver=systemd
```vim
Environment="KUBELET_KUBECONFIG_ARGS=--kubeconfig=/etc/kubernetes/kubelet.conf --require-kubeconfig-=true --cgroup-dri-=systemd"
Environment="KUBELET_SYSTEM_PODS_ARGS=--pod-manfest-path=/etc/kubernetes/manifests --allow-priviLeged=true"
Environment="KUBELET_NETWORK_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_DNS_ARGS=--cluster-dns=10.96.0.10 --cluster-domain=cluster.local"
Environment="KUBELET_AUTHZ_ARGS=--authorization-mode-Webhook --client-ca-file=/etc/kubernetes/pki/ca.crt"
Execstart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_SYSTEM_PODS_ARGS $KUBELET_NETWORK_ARGS $KUBELET_DNS_ARGS $KUBELET_AUTHZ_ARGS $KUBELET_EXTRA_ARGS
```
或者：
```vim
cat > /var/lib/kubelet/config.yaml <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```

或者：
```bash
# 配置kubelet使用国内pause镜像
# 配置kubelet的cgroups
# 获取docker的cgroups
$ DOCKER_CGROUPS=$(docker info | grep 'Cgroup' | cut -d' ' -f3)
$ echo $DOCKER_CGROUPS

$ cat >/etc/sysconfig/kubelet<<EOF
KUBELET_CGROUP_ARGS="--cgroup-driver=$DOCKER_CGROUPS"
KUBELET_EXTRA_ARGS="--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
EOF

# 启动
$ systemctl daemon-reload
$ systemctl enable kubelet && systemctl restart kubelet
```
或者：
```bash
DOCKER_CGROUPS=$(docker info | grep 'Cgroup' | cut -d' ' -f3)
echo $DOCKER_CGROUPS
cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=$DOCKER_CGROUPS --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
EOF

### 启动
$ systemctl daemon-reload
$ systemctl enable kubelet && systemctl restart kubelet
```

### ymal检测
https://www.bejson.com/validators/yaml_editor/


再次执行kubeadm init时，我发现kubeadm将cgroupDriver的配置到了`/var/lib/kubelet/kubeadm-flags.env`
后续检查/var/lib/kubelet/config.yaml 发现，里边已经被新的配置替换掉了；


### 问题
[K8S线上集群排查，实测排查Node节点NotReady异常状态](https://www.cnblogs.com/fenjyang/p/14417494.html)

### 问题
The connection to the server localhost:8080 was refused - did you specify the right host or port?
原因：kubernetes master没有与本机绑定，集群初始化的时候没有绑定，此时设置在本机的环境变量即可解决问题。
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> /etc/profile

### 问题
cni.go:239] "Unable to update cni config" err="no networks found in /etc/cni/net.d"
kubelet.go:2332] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"
解：
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml


### 问题
Kubernetes 之 Nameserver limits were exceeded  
[解：](https://www.cnblogs.com/cuishuai/p/10980852.html)
检查/etc/resolv.conf里面的nameserver肯定超过3个

当然一般被忽略掉的那个nameserver不影响服务使用的话，可以不作为紧急处理。

可以在kubelet设置一个kubermetes专用的resolv.conf文件，保证kubernetes使用到的nameserver不超过三个，这样就可以解决。
在/var/lib/kubelet路径下，修改config.yaml
resolvConf: /etc/resolv.conf
重启kubelet生效。
