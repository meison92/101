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

```
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


kubectl version --client 查看版本

#### coredns问题
```
# docker pull registry.aliyuncs.com/google_containers/coredns:<version>
# docker tag registry.aliyuncs.com/google_containers/coredns:<version> k8s.gcr.io/coredns:v<version>
# docker rmi registry.aliyuncs.com/google_containers/coredns:<version>
```

#### 获取部署所需的镜像版本
```
kubeadm config images list
kubeadm config images list --kubernetes-version=v1.22.1
拉取镜像
kubeadm config images pull
```



## start
```
kubeadm init \
 --image-repository registry.aliyuncs.com/google_containers \
 --kubernetes-version v1.22.1 \
 --apiserver-advertise-address=10.64.4.50 \
 --ignore-preflight-errors=all --v=6
```
## Post run
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf
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
```
DOCKER_CGROUPS=$(docker info | grep 'Cgroup' | cut -d' ' -f3)
echo $DOCKER_CGROUPS
cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--cgroup-driver=$DOCKER_CGROUPS --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
EOF

# 启动
$ systemctl daemon-reload
$ systemctl enable kubelet && systemctl restart kubelet
```


再次执行kubeadm init时，我发现kubeadm将cgroupDriver的配置到了`/var/lib/kubelet/kubeadm-flags.env`
后续检查/var/lib/kubelet/config.yaml 发现，里边已经被新的配置替换掉了；


### 问题：
[K8S线上集群排查，实测排查Node节点NotReady异常状态](https://www.cnblogs.com/fenjyang/p/14417494.html)