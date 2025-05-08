# -rancher-k8s-
基于rancher部署k8s 
一、基础环境说明
节点名 节点ip 角色 操作系统
node1 10.42.8.13 control-plane,etcd,master CentOS7.9
node2 10.42.8.14 control-plane,etcd,master CentOS7.9
node3 10.42.8.15 control-plane,etcd,master CentOS7.9
二、k8s节点机基础环境设置

1、设置 hostname（三台节点分别执行）

node1
hostnamectl set-hostname node1

node2
hostnamectl set-hostname node2

node3
hostnamectl set-hostname node3

2、设置 /etc/hosts（三台节点都执行）

cat < /etc/hosts
10.42.8.13 node1
10.42.8.14 node2
10.42.8.15 node3
EOF

3、设置 iptables（三台节点都执行）

iptables -P FORWARD ACCEPT

4、关闭 swap（三台节点都执行）

swapoff -a

防止开机自动挂载 swap 分区（将 /etc/fstab 中的内容注释掉）
sed -i '/ swap / s/^(.*)$/#\1/g' /etc/fstab

5、关闭 selinux（三台节点都执行）

将 selinux 由 enforcing 改为 disabled 。注意下面命令中 disabled 前面是数字 1，不是小写的英文字母 l
sed -ri 's#(SELINUX=).*#\1disabled#' /etc/selinux/config

临时关闭 enforce
setenforce 0

验证 enforce 是否已关闭：如关闭，应返回结果：Disabled
getenforce

6、将 NetworkManager 配置为忽略 calico/flannel 相关的网络接口（三台节点都执行）

cat > /etc/NetworkManager/conf.d/rke2-canal.conf <<EOF
[keyfile]
unmanaged-devices=interface-name:cali;interface-name:flannel
EOF

6、关闭并禁用firewalld和NetworkManager（三台节点都执行）

systemctl stop firewalld && systemctl disable firewalld
systemctl stop NetworkManager && systemctl disable NetworkManager

7、修改内核参数（三台节点都执行）

cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.max_map_count = 262144
vm.swappiness=0
EOF
三、rke2部署k8s

参考文档：https://ranchermanager.docs.rancher.com/zh/how-to-guides/new-user-guides/kubernetes-cluster-setup/rke2-for-rancher

1、配置rke2配置文件

说明：token可自定义字符串，但注意所有节点的token要一样

在node1上执行

mkdir -p /etc/rancher/rke2/
cat > /etc/rancher/rke2/config.yaml <<EOF
token: K108a91d54650d8ggiegn923ijgg2a9be9d1483fc932e2221f0750f14::server:57f9a32eee46999c96e936927d09af9b
tls-san: 10.42.8.13
system-default-registry: "registry.cn-hangzhou.aliyuncs.com"
cluster-cidr: 10.251.0.0/16
service-cidr: 10.252.0.0/16

ingress:
provider: nginx
options:
config-snipper: |
# 启用 snippet
enable-snippet-annotation: "true"
EOF

在node2和node3上执行

mkdir -p /etc/rancher/rke2/
cat > /etc/rancher/rke2/config.yaml <<EOF
server: https://10.42.8.13:9345
token: K108a91d54650d8ggiegn923ijgg2a9be9d1483fc932e2221f0750f14::server:57f9a32eee46999c96e936927d09af9b
tls-san: 10.42.8.13
system-default-registry: "registry.cn-hangzhou.aliyuncs.com"
cluster-cidr: 10.251.0.0/16
service-cidr: 10.252.0.0/16
EOF

2、安装并启用rke2（三个节点都执行）。特别说明：需要等node1启动成功rke2-server后，才能在node2和node3上执行

curl -sfL https://rancher-mirror.rancher.cn/rke2/install.sh | INSTALL_RKE2_MIRROR=cn INSTALL_RKE2_VERSION=v1.25.12+rke2r1 sh -
systemctl enable rke2-server && systemctl start rke2-server

3、软连接集群配置文件和操作工具（只需在node1上执行）

ln -s /var/lib/rancher/rke2/bin/kubectl /usr/local/bin/kubectl
ln -s /var/lib/rancher/rke2/bin/crictl /usr/local/bin/crictl
ln -s /var/lib/rancher/rke2/bin/ctr /usr/local/bin/ctr
ln -s /etc/rancher/rke2/rke2.yaml ~/.kube/config

4、设置crictl的默认socket（只需在node1上执行）

说明：这里是rke2的一个bug，默认设置的socket是unix:///run/containerd/containerd.sock，而实际的确是：unix:///run/k3s/containerd/containerd.sock

cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/k3s/containerd/containerd.sock
image-endpoint: unix:///run/k3s/containerd/containerd.sock
timeout: 10
debug: false
EOF

5、crt工具使用时，先设置alias别名

说明：这里是rke2的一个bug，默认设置的socket是unix:///run/containerd/containerd.sock，而实际的确是：unix:///run/k3s/containerd/containerd.sock

cat >> ~/.bashrc <<EOF
alias ctr='ctr --address=/run/k3s/containerd/containerd.sock'
EOF

source ~/.bashrc

6、至此，k8s部署完成，检查node和pod是否正常运行

[root@node1 ~]# kubectl get no
NAME STATUS ROLES AGE VERSION
node1 Ready control-plane,etcd,master 22d v1.25.12+rke2r1
node2   Ready control-plane,etcd,master 21d v1.25.12+rke2r1
node3   Ready control-plane,etcd,master 21d v1.25.12+rke2r1
[root@node1 ~]#
[root@node1 ~]# kubectl get pod -A
NAMESPACE NAME READY STATUS RESTARTS AGE
kube-system cloud-controller-manager-rke2-server-1 1/1 Running 0 2m28s
kube-system cloud-controller-manager-rke2-server-2 1/1 Running 0 61s
kube-system cloud-controller-manager-rke2-server-3 1/1 Running 0 49s
kube-system etcd-rke2-server-1 1/1 Running 0 2m13s
kube-system etcd-rke2-server-2 1/1 Running 0 87s
kube-system etcd-rke2-server-3 1/1 Running 0 56s
kube-system helm-install-rke2-canal-hs6sx 0/1 Completed 0 2m17s
kube-system helm-install-rke2-coredns-xmzm8 0/1 Completed 0 2m17s
kube-system helm-install-rke2-ingress-nginx-flwnl 0/1 Completed 0 2m17s
kube-system helm-install-rke2-metrics-server-7sggn 0/1 Completed 0 2m17s
kube-system kube-apiserver-rke2-server-1 1/1 Running 0 116s
kube-system kube-apiserver-rke2-server-2 1/1 Running 0 66s
kube-system kube-apiserver-rke2-server-3 1/1 Running 0 48s
kube-system kube-controller-manager-rke2-server-1 1/1 Running 0 2m30s
kube-system kube-controller-manager-rke2-server-2 1/1 Running 0 57s
kube-system kube-controller-manager-rke2-server-3 1/1 Running 0 42s
kube-system kube-proxy-rke2-server-1 1/1 Running 0 2m25s
kube-system kube-proxy-rke2-server-2 1/1 Running 0 59s
kube-system kube-proxy-rke2-server-3 1/1 Running 0 85s
kube-system kube-scheduler-rke2-server-1 1/1 Running 0 2m30s
kube-system kube-scheduler-rke2-server-2 1/1 Running 0 57s
kube-system kube-scheduler-rke2-server-3 1/1 Running 0 42s
kube-system rke2-canal-b9lvm 2/2 Running 0 91s
kube-system rke2-canal-khwp2 2/2 Running 0 2m5s
kube-system rke2-canal-swfmq 2/2 Running 0 105s
kube-system rke2-coredns-rke2-coredns-547d5499cb-6tvwb 1/1 Running 0 92s
kube-system rke2-coredns-rke2-coredns-547d5499cb-rdttj 1/1 Running 0 2m8s
kube-system rke2-coredns-rke2-coredns-autoscaler-65c9bb465d-85sq5 1/1 Running 0 2m8s
kube-system rke2-ingress-nginx-controller-69qxc 1/1 Running 0 52s
kube-system rke2-ingress-nginx-controller-7hprp 1/1 Running 0 52s
kube-system rke2-ingress-nginx-controller-x658h 1/1 Running 0 52s
kube-system rke2-metrics-server-6564db4569-vdfkn 1/1 Running 0 66s
四、安装 Rancher Helm Chart（仅在node1上执行）

1、安装helm

curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

2、添加 Helm Chart 仓库

helm repo add rancher-stable https://releases.rancher.com/server-charts/stable

3、为 Rancher 创建命名空间

kubectl create namespace cattle-system

4、添加rancher所需的secret，注意秘钥文件路径，可自定义，需要先把秘钥文件上传到对应的目录下

kubectl -n cattle-system create secret tls tls-rancher-ingress --cert=/opt/work/ssl_cert/skyeye.crt --key=/opt/work/ssl_cert/skyeye.key

5、通过 Helm 安装 Rancher

helm install rancher rancher-stable/rancher --namespace cattle-system --version v2.7.5 --set hostname=rancher.platform.com --set bootstrapPassword=admin --set ingress.tls.source=secret

6、至此，rancher部署完成，rancher登录页为：https://rancher.platform.com
