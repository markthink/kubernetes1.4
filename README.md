# Ansible+Vagrant本地部署Kubernetes1.4

上篇介绍了Kubernetes1.4在阿里云美西节点的部署过程，由于国内网络问题，很多留言表示并不通用，因此才有此下篇介绍本地部署K8S1.4的具体方法。

Ansible是一个简单的自动化运维工具，主要用于配置管理和应用部署，功能类似于目前业界的配置管理工具 Chef,Puppet,Saltstack。Ansible 是通过 Python 语言开发。Ansible 平台由 Michael DeHaan 创建，他同时也是知名软件 Cobbler 与 Func 的作者。

Ansible 的第一个版本发布于 2012 年 2 月。Ansible 默认通过 SSH 协议管理机器，所以 Ansible 不需要安装客户端程序在服务器上。您只需要将 Ansible 安装在一台服务器，在 Ansible 安装完后，您就可以去管理控制其它服务器。不需要为它配置数据库，Ansible 不会以 daemons 方式来启动或保持运行状态，因此使用非常方便，所以这里用它来部署K8S1.4测试环境，具体来讲Ansible 可以实现以下目标：

- 自动化部署应用
- 自动化管理配置
- 自动化的持续交付
- 自动化的（AWS）云服务管理。

从1.3开始支持新的资源类型[DaemonSet](https://github.com/eBay/Kubernetes/blob/master/docs/design/daemon.md),kube-scheduler,kube-apiserver,kube-controller-manager,kube-proxy,kube-discovery都已经放入POD中，使用更加方便了，上文提到Kubernetes1.4的新功能是引入了kubeadm部署机制(暂时还是alpha版本)，简化了Kubernetes集群的构建，我们部署一个K8S集群只要如下四个步骤，通过Ansible部署会更加简单，只要二步就OK。

- 安装docker、kubelet、kubectl、kubeadm
  - docker 容器运行环境
  - kubelet 集群最核心组件，它运行在所有集群中的机器，并实际操作POD和容器
  - kubectl 交互命令行控制集群
  - kubeadm Kubernetes1.4新增，替换之前的kube-up.sh脚本，用于集群的创建和节点的增加
- kubeadm init初始化master
- kubeadm join --token <token-id>  <master-ip>
- 部署POD网络

之前和网友交流过，有些网友已经本地部署成功了，由于比较忙，所以这篇本地部署文章让大家久等了，很抱歉，具体部署代码可以从我的github下载，下面开始我们的本地部署过程。

## 准备

准备三台虚拟机

- 硬件配置： 三台 CPU1核 内存1.5G机器-需要可以访问国内网络(Ansible安装和国内镜像下载需要)
- 操作系统：Centos7.2
- 代码库-https://github.com/MarkThink/kubernetes1.4

注:本代码库源于https://github.com/errordeveloper/kubernetes-ansible-vagrant.git

Vagrant配置文件Vagrantfile：

```
# -*- mode: ruby -*-
# # vi: set ft=ruby :

boxes = {
  ubuntu: "ubuntu/xenial64",
  centos: "centos7.2",
}

distro = :centos # :ubuntu

Vagrant.configure(2) do |config|

  (1..3).each do |i|
    config.vm.define "k8s#{i}" do |s|
      s.ssh.forward_agent = true
      s.vm.box = boxes[distro]
      s.vm.hostname = "k8s#{i}"
      s.vm.provision :shell, path: "scripts/bootstrap_ansible_#{distro.to_s}.sh"
      n = 10 + i
      s.vm.network "private_network", ip: "172.42.42.#{n}", netmask: "255.255.255.0",
        auto_config: true,
        virtualbox__intnet: "k8s-net"
      s.vm.provider "virtualbox" do |v|
        v.name = "k8s#{i}"
        v.memory = 1536
        v.gui = false
      end
    end
  end

  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end
end
```

启动虚拟机

```sh
vagrant up
//进入虚拟机环境
vagrant ssh k8s1
vagrant ssh k8s2
vagrant ssh k8s3
#切换超级用户
su
#关闭SE
setenforce 0
#关闭firewalld
systemctl stop firewalld && systemctl disable firewalld
#进入三台虚拟机，更改IP地址为静态方式
vi /etc/sysconfig/network-scripts/ifcfg-enp0s8
BOOTPROTO=static
#重启网络服务,使三台机器可以两两互通
systemctl restart network
```

将此文件yum.repo复制到/etc/yum.repo.d/目录，准备Kubernetes安装的软件包。

```
cp yum.repo /etc/yum.repo.d/
```

```sh
#yum.repo
[base]
name=base-repo
baseurl=http://yum.51yixiao.com/base/
enabled=1
gpgcheck=0
gpgkey=http://yum.51yixiao.com/Centos7Base/RPM-GPG-KEY-CentOS-7

[epel]
name=epel-repo
baseurl=http://yum.51yixiao.com/epel/
enabled=1
gpgcheck=0
gpgkey=http://yum.51yixiao.com/Centos7Base/RPM-GPG-KEY-CentOS-7

[extras]
name=extras-repo
baseurl=http://yum.51yixiao.com/extras/
enabled=1
gpgcheck=0
gpgkey=http://yum.51yixiao.com/Centos7Base/RPM-GPG-KEY-CentOS-7

[kubernetes]
name=kubernetes-repo
baseurl=http://yum.51yixiao.com/kubernetes/
enabled=1
gpgcheck=0
gpgkey=http://yum.51yixiao.com/Centos7Base/RPM-GPG-KEY-CentOS-7

[updates]
name=updates-repo
baseurl=http://yum.51yixiao.com/updates/
enabled=1
gpgcheck=0
gpgkey=http://yum.51yixiao.com/Centos7Base/RPM-GPG-KEY-CentOS-7
```
##开始Ansible部署

```sh
vagrant ssh k8s1
vagrant ssh k8s2
vagrant ssh k8s3
#切换超级用户
su
#使用Ansible安装K8S基础环境
ansible-playbook /vagrant/ansible/k8s-base.yml -c local
```

这会完成docker、kubelet、kubectl、kubeadm的安装,日志如下：

```sh
 [WARNING]: provided hosts list is empty, only localhost is available


PLAY [localhost] ***************************************************************

TASK [setup] *******************************************************************
ok: [localhost]

TASK [k8s-base : Ensure SSH Directories] ***************************************
skipping: [localhost]

TASK [k8s-base : Copy SSH Key Files] *******************************************
skipping: [localhost] => (item=id_rsa)
skipping: [localhost] => (item=id_rsa.pub)
skipping: [localhost] => (item=config)

TASK [k8s-base : Ensure Authorized SSH Key] ************************************
skipping: [localhost]

TASK [k8s-base : Remove Default Host Entry] ************************************
changed: [localhost]

TASK [k8s-base : Ensure Hosts File] ********************************************
changed: [localhost] => (item={u'ip': u'172.42.42.11', u'name': u'k8s1'})
changed: [localhost] => (item={u'ip': u'172.42.42.12', u'name': u'k8s2'})
changed: [localhost] => (item={u'ip': u'172.42.42.13', u'name': u'k8s3'})

TASK [k8s-base : Ensure Kubernetes APT Key] ************************************
skipping: [localhost]

TASK [k8s-base : Ensure Kubernetes APT Repository] *****************************
skipping: [localhost]

TASK [k8s-base : Ensure Base Kubernetes] ***************************************
skipping: [localhost] => (item=[])

TASK [k8s-base : file] *********************************************************
changed: [localhost]

TASK [k8s-base : Ensure Base Kubernetes] ***************************************
changed: [localhost] => (item=[u'docker', u'kubelet', u'kubeadm', u'kubectl', u'kubernetes-cni'])

TASK [k8s-base : Ensure docker.service] ****************************************
changed: [localhost]

TASK [k8s-base : Ensure kubelet.service] ***************************************
changed: [localhost]

TASK [k8s-base : Ensure firewalld.service] *************************************
changed: [localhost]

TASK [k8s-base : firewalld] ****************************************************
changed: [localhost]

TASK [k8s-base : firewalld] ****************************************************
changed: [localhost]

TASK [k8s-base : firewalld] ****************************************************
changed: [localhost]

TASK [k8s-base : firewalld] ****************************************************
changed: [localhost]

TASK [k8s-base : firewalld] ****************************************************
changed: [localhost]

TASK [k8s-base : command] ******************************************************
changed: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=14   changed=13   unreachable=0    failed=0
```

由于网络问题，这里已经准备了相应的镜像

```sh
docker pull registry.51yixiao.com/google_containers/kube-controller-manager-amd64:v1.4.0
docker pull registry.51yixiao.com/google_containers/kube-proxy-amd64:v1.4.0
docker pull registry.51yixiao.com/google_containers/kube-apiserver-amd64:v1.4.0
docker pull registry.51yixiao.com/google_containers/kube-scheduler-amd64:v1.4.0
docker pull registry.51yixiao.com/google_containers/kube-discovery-amd64:1.0
docker pull registry.51yixiao.com/google_containers/kubedns-amd64:1.7
docker pull registry.51yixiao.com/google_containers/exechealthz-amd64:1.1
docker pull registry.51yixiao.com/google_containers/kube-dnsmasq-amd64:1.3
docker pull registry.51yixiao.com/google_containers/pause-amd64:3.0
docker pull registry.51yixiao.com/google_containers/etcd-amd64:2.2.5
```

镜像前缀修改

```sh
docker tag registry.51yixiao.com/google_containers/kube-controller-manager-amd64:v1.4.0 gcr.io/google_containers/kube-controller-manager-amd64:v1.4.0
```

```
docker tag registry.51yixiao.com/google_containers/kube-proxy-amd64:v1.4.0 gcr.io/google_containers/kube-proxy-amd64:v1.4.0
```

```
docker tag registry.51yixiao.com/google_containers/kube-apiserver-amd64:v1.4.0 gcr.io/google_containers/kube-apiserver-amd64:v1.4.0
```

```
docker tag registry.51yixiao.com/google_containers/kube-scheduler-amd64:v1.4.0 gcr.io/google_containers/kube-scheduler-amd64:v1.4.0
```

```
docker tag registry.51yixiao.com/google_containers/kube-discovery-amd64:1.0 gcr.io/google_containers/kube-discovery-amd64:1.0
```

```
docker tag registry.51yixiao.com/google_containers/kubedns-amd64:1.7 gcr.io/google_containers/kubedns-amd64:1.7
```

```
docker tag registry.51yixiao.com/google_containers/exechealthz-amd64:1.1 gcr.io/google_containers/exechealthz-amd64:1.1
```

```
docker tag registry.51yixiao.com/google_containers/kube-dnsmasq-amd64:1.3 gcr.io/google_containers/kube-dnsmasq-amd64:1.3
```

```
docker tag registry.51yixiao.com/google_containers/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0
```

```
docker tag registry.51yixiao.com/google_containers/etcd-amd64:2.2.5 gcr.io/google_containers/etcd-amd64:2.2.5
```

准备POD网络镜像

```sh
docker pull registry.51yixiao.com/weaveworks/weave-kube:1.7.2
docker pull registry.51yixiao.com/weaveworks/weave-npc:1.7.2
```

镜像前缀修改

```sh
docker tag registry.51yixiao.com/weaveworks/weave-kube:1.7.2 weaveworks/weave-kube:1.7.2
docker tag registry.51yixiao.com/weaveworks/weave-npc:1.7.2 weaveworks/weave-npc:1.7.2
```

Master节点

```
gcr.io/google_containers/kube-controller-manager-amd64:v1.4.0
gcr.io/google_containers/kube-proxy-amd64:v1.4.0
gcr.io/google_containers/kube-apiserver-amd64:v1.4.0
gcr.io/google_containers/kube-scheduler-amd64:v1.4.0
gcr.io/google_containers/kube-discovery-amd64:1.0
gcr.io/google_containers/kubedns-amd64:1.7
gcr.io/google_containers/exechealthz-amd64:1.1
gcr.io/google_containers/kube-dnsmasq-amd64:1.3
gcr.io/google_containers/pause-amd64:3.0
gcr.io/google_containers/etcd-amd64:2.2.5

weaveworks/weave-kube:1.7.2
weaveworks/weave-npc:1.7.2
```

Node节点

```
gcr.io/google_containers/kube-proxy-amd64:v1.4.0
gcr.io/google_containers/pause-amd64:3.0
weaveworks/weave-kube:1.7.2
weaveworks/weave-npc:1.7.2
```

查看镜像数量(Master节点10个镜像 Node节点2个镜像)

```sh
docker images|grep gcr.io|wc -l
```

###Setup1.开始部署Master节点

这里以K8S1节点作为Master节点，执行下面的命令，完成Master部署

```sh
ansible-playbook /vagrant/ansible/k8s-master.yml -c local
```

日志信息：

```sh
 [WARNING]: provided hosts list is empty, only localhost is available


PLAY [localhost] ***************************************************************

TASK [setup] *******************************************************************
ok: [localhost]

TASK [k8s-master : Ensure kubeadm initialization] ******************************
changed: [localhost]

TASK [k8s-master : Ensure Network Start Script] ********************************
ok: [localhost] => (item=start-weave)
ok: [localhost] => (item=start-calico)
ok: [localhost] => (item=start-canal)

TASK [k8s-master : Ensure jq package is installed] *****************************
skipping: [localhost] => (item=[])

TASK [k8s-master : Ensure jq package is installed] *****************************
changed: [localhost] => (item=[u'jq'])

TASK [k8s-master : Set --advertise-address flag in kube-apiserver static pod manifest (workaround for https://github.com/kubernetes/kubernetes/issues/34101)] ***
changed: [localhost]

TASK [k8s-master : Set --cluster-cidr flag in kube-proxy daemonset (workaround for https://github.com/kubernetes/kubernetes/issues/34101)] ***
changed: [localhost]

TASK [k8s-master : firewalld] **************************************************
changed: [localhost]

TASK [k8s-master : firewalld] **************************************************
changed: [localhost]

TASK [k8s-master : command] ****************************************************
changed: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=9    changed=7    unreachable=0    failed=0
```
生成的配置文件列表

```
# 证书
/etc/kubernetes/pki/apiserver-key.pem
/etc/kubernetes/pki/apiserver.pem
/etc/kubernetes/pki/apiserver-pub.pem
/etc/kubernetes/pki/ca-key.pem
/etc/kubernetes/pki/ca.pem
/etc/kubernetes/pki/ca-pub.pem
/etc/kubernetes/pki/sa-key.pem
/etc/kubernetes/pki/sa-pub.pem
/etc/kubernetes/pki/tokens.csv
# Master配置
/etc/kubernetes/manifests/kube-scheduler.json
/etc/kubernetes/manifests/kube-controller-manager.json
/etc/kubernetes/manifests/kube-apiserver.json
/etc/kubernetes/manifests/etcd.json
# kubelet配置
/etc/kubernetes/admin.conf
/etc/kubernetes/kubelet.conf
```
```
#查看启动配置
ps aux|grep kubelet
```

```sh
#输出日志
/usr/bin/kubelet --kubeconfig=/etc/kubernetes/kubelet.conf --require-kubeconfig=true --pod-manifest-path=/etc/kubernetes/manifests --allow-privileged=true --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --cluster-dns=100.64.0.10 --cluster-domain=cluster.local --v=4
```

由于POD网络未配置，kube-dns未启动

```
[root@k8s1 files]# kubectl get po --namespace=kube-system
NAME                             READY     STATUS              RESTARTS   AGE
etcd-k8s1                        1/1       Running             0          5m
kube-apiserver-k8s1              1/1       Running             3          5m
kube-controller-manager-k8s1     1/1       Running             1          6m
kube-discovery-982812725-uz406   1/1       Running             0          6m
kube-dns-2247936740-s412w        0/3       ContainerCreating   0          6m
kube-proxy-amd64-2228u           1/1       Running             0          2m
kube-proxy-amd64-3um33           1/1       Running             0          6m
kube-proxy-amd64-945s5           1/1       Running             0          2m
kube-scheduler-k8s1              1/1       Running             1          5m
weave-net-i4qk7                  2/2       Running             0          28s
weave-net-k12m3                  2/2       Running             0          28s
weave-net-vh456                  2/2       Running             0          28s
```

###Setup2.添加work节点

k8s2/k8s3设置为工作节点，执行下面的命令

```
ansible-playbook /vagrant/ansible/k8s-worker.yml -c local
```

```sh
[root@k8s2 ~]# ansible-playbook /vagrant/ansible/k8s-worker.yml -c local
 [WARNING]: provided hosts list is empty, only localhost is available


PLAY [localhost] ***************************************************************

TASK [setup] *******************************************************************
ok: [localhost]

TASK [k8s-worker : Join Kubernetes Cluster] ************************************
changed: [localhost]

PLAY RECAP *********************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0
```

查看节点状态

```
[root@k8s1 files]# kubectl get node
NAME      STATUS    AGE
k8s1      Ready     49m
k8s2      Ready     45m
k8s3      Ready     45m
```

###Setup3.部署Pod网络

Kubernetes1.2版本默认使用的是flannel网络，用于解决POD跨主机之间的通信。新版本未提供默认的网络插件，在部署应用集群之前，必须要配置POD网络。

未配置POD网络，默认的KUBE-DNS是无法启动的。这里使用[weave](https://www.weave.works/products/weave-net/)网络方案,当然也可以使用[Calico](https://github.com/projectcalico/calico-containers/tree/master/docs/cni/kubernetes/manifests/kubeadm) 或 [Canal](https://github.com/tigera/canal/tree/master/k8s-install/kubeadm)。

在Master节点执行下面的命令安装POD网络

```sh
[root@k8s1 files]# pwd
/vagrant/ansible/roles/k8s-master/files
[root@k8s1 files]# ./start-weave
```

查看部署POD

```
[root@k8s1 files]# kubectl get po --namespace=kube-system
NAME                             READY     STATUS    RESTARTS   AGE
etcd-k8s1                        1/1       Running   0          54m
kube-apiserver-k8s1              1/1       Running   3          55m
kube-controller-manager-k8s1     1/1       Running   1          55m
kube-discovery-982812725-uz406   1/1       Running   0          55m
kube-dns-2247936740-s412w        3/3       Running   1          55m
kube-proxy-amd64-2228u           1/1       Running   0          51m
kube-proxy-amd64-3um33           1/1       Running   0          55m
kube-proxy-amd64-945s5           1/1       Running   0          51m
kube-scheduler-k8s1              1/1       Running   1          54m
weave-net-i4qk7                  2/2       Running   0          49m
weave-net-k12m3                  2/2       Running   0          49m
weave-net-vh456                  2/2       Running   0          49m
```

至此完成K8S1.4无网环境的部署，希望此文对大家有帮助。



## Happly Enjoy!

