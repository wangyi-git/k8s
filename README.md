# k8s
kubeadm部署k8s集群以及dashboard页面
hero小执着
0人评论
30416人阅读
2019-08-20 12:58:51

一.部署环境（所有节点都做）
OS：CentOS Linux release 7.6.1810 (Core)
kernel： 3.10.0-957.21.3.el7.x86_64
1 台master 2台node
环境准备（三台机器都做以下操作）

1.1 关闭防火墙：
systemctl stop firewalld
systemctl disable firewalld

1.2 关闭selinux：
sed -i 's/enforcing/disabled/' /etc/selinux/config 
setenforce 0

1.3 关闭swap：
swapoff -a  # 临时
sed -i 's/.*swap.*/#&/' /etc/fstab  # 永久

1.4 确保iptables可用：
echo 1 > /proc/sys/net/bridge/bridge-nf-call-ip6tables 
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables

1.5 添加主机名与IP对应关系：（三台机器都做）
214.92 ： hostnamectl set-hostname k8s-master && bash  #修改主机名并立即生效
214.97 ： hostnamectl set-hostname k8s-node1 && bash
214.98 ： hostnamectl set-hostname k8s-node2 && bash
cat /etc/hosts
192.168.214.92 k8s-master   #(ip根据自身实际情况修改)
192.168.214.97 k8s-node1    #(ip根据自身实际情况修改)
192.168.214.98 k8s-node2    #(ip根据自身实际情况修改)
https://s1.51cto.com/images/blog/201909/19/ecbc71fa6f6535d9762e01c590fd45ea.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_100,g_se,x_10,y_10,shadow_90,type_ZmFuZ3poZW5naGVpdGk=
kubeadm部署k8s集群以及dashboard页面

1.6 配置国内yum源：
 yum install -y wget

 mkdir /etc/yum.repos.d/bak && mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak

 wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.cloud.tencent.com/repo/centos7_base.repo

 wget -O /etc/yum.repos.d/epel.repo http://mirrors.cloud.tencent.com/repo/epel-7.repo

 yum clean all && yum makecache

1.7 配置kubenetes yum源

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg         https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

所有设备均执行如下命令
二.安装docker和kube组件
yum install -y kubelet-1.15.0 kubeadm-1.15.0 kubectl-1.15.0 docker-ce

设置开机自启
systemctl enable docker 
systemctl enable kuhbelet

2.1 docker架构关系图
kubeadm部署k8s集群以及dashboard页面

2.2 开启docker和kubelet（所有节点）
systemctl start docker
 systemctl start kubelet

2.3 下载所需docker镜像(所有节点均做以下操作)

docker pull mirrorgooglecontainers/kube-apiserver-amd64:v1.15.0 
docker pull mirrorgooglecontainers/kube-controller-manager-amd64:v1.15.0 
docker pull mirrorgooglecontainers/kube-scheduler-amd64:v1.15.0 
docker pull mirrorgooglecontainers/kube-proxy-amd64:v1.15.0 
docker pull mirrorgooglecontainers/pause:3.1 
docker pull mirrorgooglecontainers/etcd:3.3.10 
docker pull coredns/coredns:1.3.1 

2.4 镜像打标
docker tag docker.io/mirrorgooglecontainers/kube-apiserver-amd64:v1.15.0 k8s.gcr.io/kube-apiserver:v1.15.0 
docker tag docker.io/mirrorgooglecontainers/kube-scheduler:v1.15.0 k8s.gcr.io/kube-scheduler:v1.15.0 
docker tag docker.io/mirrorgooglecontainers/kube-controller-manager:v1.15.0 k8s.gcr.io/kube-controller-manager:v1.15.0 
docker tag docker.io/mirrorgooglecontainers/kube-proxy-amd64:v1.15.0  k8s.gcr.io/kube-proxy:v1.15.0
docker tag mirrorgooglecontainers/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10 
docker tag mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1 
docker tag coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1

2.5 忽略swap报错
[root@master ~]# vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--fail-swap-on=false"

2.6 初始化master（只做master节点）安装
 kubeadm init  --kubernetes-version=v1.15.0 --ignore-preflight-errors=Swap  --ignore-preflight-errors=Numcpu --pod-network-cidr 10.244.0.0/16
#指定版本，忽略swap和cpu报错
#10.244.0.0/16为pod网络  
初始化集群出错时，需要所有节点清掉这些数据.可用以下命令
kubeadm reset
systemctl stop kubelet
 docker rm -f -v $(docker ps  -a -q)
 rm -rf /etc/kubernetes
 rm -rf  /var/lib/etcd
rm -rf   /var/lib/kubelet
  rm -rf  $HOME/.kube/config
 iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
 yum reinstall -y kubelet
 systemctl daemon-reload
  systemctl restart docker

2.7 kubectl 客户端用户配置文件配置
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

2.8 记录安装后输出，供后面添加node节点使用

kubeadm部署k8s集群以及dashboard页面

2.9 安装flannel网络
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

2.10 加入master（在node节点执行）
复制master节点刚才输出的token（在两个节点分别执行，）
[root@node1 ~]#kubeadm join 192.168.214.92:6443 --token ltsy9m.gscdwrfr12klwdb0     --discovery-token-ca-cert-hash sha256:b7aaaedb8df894499bf01413c549e48b844e20c15a91d2e174447dfcad88ba5b --ignore-preflight-errors=all

kubeadm部署k8s集群以及dashboard页面

[root@node2 ~]# kubeadm join 192.168.214.92:6443 --token gdpagl.n2vzci4kh4zdy4s1 --discovery-token-ca-cert-hash sha256:261d86818a0271eb997e5a9f5ef1ced0585b526fefdef0c18a53884b3b02bdf2 --ignore-preflight-errors=all
kubeadm部署k8s集群以及dashboard页面

2.11 node节点加入master失败时，需要清除数据重新reset，删除cnio网络，可用以下命令:
kubeadm reset
systemctl stop kubelet
docker rm -f -v $(docker ps  -a -q)
rm -rf /etc/kubernetes
rm -rf  /var/lib/etcd
rm -rf   /var/lib/kubelet
rm -rf  $HOME/.kube/config
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
yum reinstall -y kubelet
systemctl daemon-reload
systemctl restart docker

2.12 kubectl get nodes && kubectl get cs 查看集群状态
kubeadm部署k8s集群以及dashboard页面


三.搭建dashboard（mster节点操作）
这里我已经把dashboard写好了，直接下载我的上传的你的root目录即可。
kubernetes-dashboard.yaml链接：https://pan.baidu.com/s/14hwEGoCnzOw40m17HeYjfQ

3.1 创建dasahbord

kubectl create -f /root/kubernetes-dashboard.yaml
3.2 查看pod
kubectl get pod -n kube-system -o wide
3.3 查看service（dashboard端口30000是否映射成功）
kubectl get service -n kube-system
kubeadm部署k8s集群以及dashboard页面

所有的pod状态为running即可

kubeadm部署k8s集群以及dashboard页面
3.4 浏览器访问(这里浏览器推荐火狐，其他的可能不兼容)如果用谷歌浏览器无法访问可参照此文档解决 https://www.jianshu.com/p/8021285cc37d。
https://ip:port
例如我的是：https://192.168.214.92:30000
kubeadm部署k8s集群以及dashboard页面
3.5创建admin权限账号（否则进入页面没有超级权限）
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin


3.6获取token登陆dashboard页面
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
kubeadm部署k8s集群以及dashboard页面

kubeadm部署k8s集群以及dashboard页面

4.大功告成
kubeadm部署k8s集群以及dashboard页面
©著作权归作者所有：来自51CTO博客作者hero小执着的原创作品，如需转载，请注明出处，否则将追究法律责任
