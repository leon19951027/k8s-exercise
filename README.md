# 安装


#!/bin/bash

swapoff -a
#更新为国内源

cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

#yum安装

yum -y install kubeadm-1.15.1-0 kubelet-1.15.1-0 kubectl-1.15.1-0


#拉镜像
for i in `kubeadm config images list`; do
    imageName=${i#k8s.gcr.io/}
    docker pull registry.aliyuncs.com/google_containers/$imageName
    docker tag registry.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.aliyuncs.com/google_containers/$imageName
   done;

 echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables


#下载flannel网络
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
