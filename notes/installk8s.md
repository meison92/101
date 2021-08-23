
## network



## firewall


### 关闭swap、关闭防火墙、禁用Selinux
```
#关闭磁盘交换  很重要必须执行
sudo swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab

ufw status
ufw disable

sudo apt install selinux-utils
setenforce 0

### why:
不止部署k8s，许多公司在装机过程就就直接关闭了swap、selinux和防火墙
selinux，这个是用来加强安全性的一个组件，但非常容易出错且难以定位，一般上来装完系统就先给禁用了

iptables防火墙，会对所有网络流量进行过滤、转发，如果是内网机器一般都会直接关闭，省的影响网络性能，但k8s不能直接关了，k8s是需要用防火墙做ip转发和修改的，当然也看使用的网络模式，如果采用的网络模式不需要防火墙也是可以直接关闭的

swap，这个当内存不足时，linux会自动使用swap，将部分内存数据存放到磁盘中，这个这样会使性能下降，为了性能考虑推荐关掉

```

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
```
/etc/apt/sources.list 添加deb 行 
deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main

执行
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt update

执行
sudo apt install -y kubelet kubeadm kubectl

阻止自动更新(apt upgrade时忽略)
sudo apt-mark hold kubelet kubeadm kubectl
```

## start
```
kubeadm init \
 --image-repository registry.aliyuncs.com/google_containers \
 --kubernetes-version <version> \
 --apiserver-advertise-address=<ipaddress>
```