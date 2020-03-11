此篇文档是个人学习如何在 MacOS 电脑上搭建一个完整的 Kubebernetes 集群，主要利用 RancherOS with Docker Machine 的方式进行。

**目前状态**：简单完成了 Master 的安装，还没有弄 Worker...


- [预备知识](#%e9%a2%84%e5%a4%87%e7%9f%a5%e8%af%86)
- [虚拟机预装](#%e8%99%9a%e6%8b%9f%e6%9c%ba%e9%a2%84%e8%a3%85)
- [Ansible 主机](#ansible-%e4%b8%bb%e6%9c%ba)
- [安装 Common 组件](#%e5%ae%89%e8%a3%85-common-%e7%bb%84%e4%bb%b6)
- [安装 Master 组件](#%e5%ae%89%e8%a3%85-master-%e7%bb%84%e4%bb%b6)
- [安装 Worker 组件](#%e5%ae%89%e8%a3%85-worker-%e7%bb%84%e4%bb%b6)
- [其它内容](#%e5%85%b6%e5%ae%83%e5%86%85%e5%ae%b9)
- [Reference](#reference)

# 预备知识

所涉及到的 Docker-Machine/Docker/Ansible/Kubernetes/Shell 知识这里不展开说明。

最早参考的文档：

- [Running K8S Cluster on RancherOS](https://medium.com/@imikushin/running-kubernetes-cluster-on-rancheros-b2bd1308eb6d)
- [jimmysong 的中文文档](https://jimmysong.io/kubernetes-handbook/concepts/)

标准的 Kubernetes 组件有：

- Common(每台都需要安装)
  - etcd
  - flannel
- Master
  - kube-apiserver
  - kube-controller-manager
  - kube-scheduler
- Worker
  - kube-proxy
  - kubelet

# 虚拟机预装

首先，我们利用 Docker Machine 创建 4 台虚拟的服务器，我这里用的 RancherOS 作为 boot2docker 的镜像

> 我这里没有深入说明 docke-machine 的用法，具体可以先从这一篇官方文章开始 [Docker Machine vs Docker Desktop for Mac](https://docs.docker.com/docker-for-mac/docker-toolbox/)

```shell
docker-machine create -d virtualbox --virtualbox-boot2docker-url ~/Downloads/rancheros.iso master-1
docker-machine create -d virtualbox --virtualbox-boot2docker-url ~/Downloads/rancheros.iso node-1
docker-machine create -d virtualbox --virtualbox-boot2docker-url ~/Downloads/rancheros.iso node-2
docker-machine create -d virtualbox --virtualbox-boot2docker-url ~/Downloads/rancheros.iso node-3
```

# Ansible 主机

利用一段命令把 docker machine 的主机导出 Ansible 识别的 Inventory 主机列表，

```shell
./bin/inventory
```

# 安装 Common 组件

安装公共服务(etcd/flannel)

```shell
export ETCD_ALL_URLS=`docker-machine ls | grep "tcp"| awk '{sub("tcp://","",$5);sub(":2376","",$5); print $1"=http://"$5":2380"}'`
export ETCD_ALL_URLS=`echo -n $ETCD_ALL_URLS | tr '\n' ','`

ansible-playbook playbook-common.yaml --inventory hosts --extra-vars "etcd_all_urls=${ETCD_ALL_URLS} flannel_network=10.1.0.0/16"
```

# 安装 Master 组件

安装 Master 服务(kube-apiserver/kube-controller/kube-scheduler)

```shell
ansible-playbook playbook-master.yaml --inventory hosts
```

安装完毕之后，就可以利用 curl 或者 kubectl 查看 API 的信息了

```shell
kubectl --token="7db2f1c02d721320" --server=https://192.168.99.107:6443 --insecure-skip-tls-verify=true cluster-info
```

更精简的方法就是把 `KUBECONFIG` 导入到全局使用

```
export KUBECONFIG
kubectl cluster-info
```

# 安装 Worker 组件

TBD

# 其它内容

设置所有的 Docker 镜像加速服务

```shell
ansible-playbook playbook-other.yaml --inventory hosts \
--extra-vars "mirror=<Registry Mirror>"
```

# Reference

[中文解释 Kuberentes 授权机制](https://zhangchenchen.github.io/2017/08/17/kubernetes-authentication-authorization-admission-control/)
