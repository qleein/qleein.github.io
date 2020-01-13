---
layout: post
title:  "CentOS 7 使用kubeadm 部署 Kubernetes"
date:   2019-03-16 14:58:00 +0800
---

### 关闭swap

执行`swapoff`临时关闭swap。

重启后会失效，若要永久关闭，可以编辑`/etc/fstab`文件，将其中swap分区一行注释掉

```
#/dev/mapper/centos-swap swap                    swap    defaults        0 0

```

### 安装配置docker

可以参考[官方安装文档](https://docs.docker.com/engine/installation/)

#### 1\. 安装docker

```
$ yum install yum-utils device-mapper-persistent-data lvm2
$ yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
$ yum update && yum install docker-ce-18.06.2.ce

```

#### 2\. docker配置

创建文件`/etc/docker/daemon.json`, 写入下面的内容。

```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}

```

若在国内部署，可以使用国内的docker源加快拉取速度，`/etc/docker/daemon.json`中加入国内源。

```
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}

```

#### 3\. 重启docker

```
mkdir -p /etc/systemd/system/docker.service.d
systemctl daemon-reload
systemctl restart docker

```

### 安装kubeadm, kubelet和kubectl

#### 1\. 添加yum仓库

如果节点在国内，可以使用国内镜像仓库，这里使用了阿里云的，创建`/etc/yum.repos.d/kubernetes.repo`，文件如下内容

```
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

```

如果在国外，可以使用谷歌官方镜像仓库，文件内容如下

```
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*

```

#### 2\. 关闭SELinux

kubelet不支持SELinux, 这里需要将SELinux设置为`permissive`模式

```
$ setenforce 0
$ sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

```

#### 3\. 安装 kubelet, kubectl和kubeadm

```
$ yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
$ systemctl enable --now kubelet

```

#### 4\. 配置sysctl

在RHEL/CentOS 7上由于 iptables 被绕过导致网络请求被错误的路由。您得保证 在您的 sysctl 配置中 net.bridge.bridge-nf-call-iptables 被设为1。

创建文件`/etc/sysctl.d/k8s.conf`， 文件内容如下

```
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

```

执行`sysctl --system`使配置生效

#### 5\. 拉取镜像

如果节点在国外，可以跳过这一步。

执行`kubeadm config images pull`查看到gcr.io的连接，如果拉取成功可以进入下一步。

如果失败，[说明无法访问gcr.io](http://xn--gcr-7z8f0hx58a2q8c9lat55e.io)。这时需要手动拉取镜像，可以执行下面的脚本，从阿里云拉取相应镜像。

```
#!/bin/bash

images=(
    kube-apiserver:v1.13.4
    kube-controller-manager:v1.13.4
    kube-scheduler:v1.13.4
    kube-proxy:v1.13.4
    pause:3.1
    etcd:3.2.24
    coredns:1.2.6
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
done

```

也可以在国外的vps上拉取相应的镜像，执行`docker save`保存为tar文件，复制到需要的节点上，`docker load`加载，然后通过`docker tag`恢复镜像的tag。

> 执行`kubeadm config images list`可以查看所需镜像

### 安装pod网络插件

这里使用flannel

```
$ yum insatll flanneld

```

### 初始化集群的master



#### 1\. kubeadm init初始化

可以先通过`kubeadm config images pull`确认镜像拉取成功。

通过`kubeadm init <args>`初始化。由于这里使用了flannel作为网络插件，在初始化时需要加入`--pod-network-cidr=10.244.0.0/16`指定网络。如果不适用默认的

```
kubeadm init --pod-network-cidr=10.244.0.0/16

```

初始化成功后，会输出下面的信息

```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <master-ip>:<master-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

```

> 注意这里最后一行的`kubeadm join .....`，在其他节点执行此行命令，使节点加入到集群中。

如果使用非root账户操作kubectl，则需执行如下命令

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

#### 2\. 部署`flannel`

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml

```

通过`kubectl get nodes`查看节点，如果看到`master`节点表示安装成功。

```
$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
host-168   Ready    master   41h   v1.13.4

```

### 将其他节点加入到集群中

只有master需要执行`kubeadm init`初始化，其他node节点通过执行`kubeadm join`加入到集群中

命令及参数采用`kubeadm init`时的输出，示例如下

```
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>

```

这里的toke有效期只有24小时，如果过期在master上执行`kubeadm create token`创建新的token。

执行`kubectl get nodes`查看node节点

```
$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
host-167   Ready    <none>   41h   v1.13.4
host-168   Ready    master   41h   v1.13.4

```

> 集群中的master和node节点的hostname不能重复，否则会加入集群失败。 如果重复，可以通过`hostnamectl set-hostname NAME`修改节点的hostname

### 安装Web界面(可选)

安装kuberenets-dashboard后可以通过浏览器查看/更改集群

#### 1\. 拉取镜像

和前述相同，需要node节点拉取下面几个镜像。国内节点需要从阿里云拉取或者从国外节点拉取后传到节点再加载

```
k8s.gcr.io/kubernetes-dashboard-amd64:v1.10.1
gcr.io/google_containers/kube-proxy-amd64:v1.9.8 
k8s.gcr.io/pause:3.1
gcr.io/google_containers/pause-amd64:3.0

```

#### 2\. 部署dashboard

获取yaml配置文件

```
$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

```

kubernetes-dashboard默认需要通过`kubectl proxy`来访问，但是这时使用的是HTTP，在HTTP下访问dashboard时即使登录成功,也会卡在登录页面。可以选择让dashboard直接监听外部端口，不经过`kubectl proxy`的代理。

为此可以在kubernets-dashboard.yaml最后一栏，`Dashboard Service`中，增加了`type: NodePort`和`nodePort: 30001`，更改后的结果如下所示。（这里30001端口也可以更改）

```
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard

部署dashboard

```

通过kubectl部署dashboard

```
$ kubectl apply -f kubernets-dashboard.yaml

```

访问`https://nodeIP:30001`，出现登录界面表示成功。这里的证书不被浏览器信任，chrome无法访问的话，可以用firefox打开。

#### 3\. 创建admin用户来访问dashboard

这里为了简单，创建了一个admin账户用来登录

新建文件`dashboard-adminuser.yaml`，内容如下

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
  
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

```

在master上执行`kubectl apply -f dashboard-adminuser.yaml`创建admin用户及role绑定。

获取token

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')

```

输出如下所示

```
Name:         admin-user-token-6gl6l
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=admin-user
              kubernetes.io/service-account.uid=b16afba9-dfec-11e7-bbb9-901b0e532516

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTZnbDZsIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiJiMTZhZmJhOS1kZmVjLTExZTctYmJiOS05MDFiMGU1MzI1MTYiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.M70CU3lbu3PP4OjhFms8PVL5pQKj-jj4RNSLA4YmQfTXpPUuxqXjiTf094_Rzr0fgN_IVX6gC4fiNUL5ynx9KU-lkPfk0HnX8scxfJNzypL039mpGt0bbe1IXKSIRaq_9VW59Xz-yBUhycYcKPO9RM2Qa1Ax29nqNVko4vLn1_1wPqJ6XSq3GYI8anTzV8Fku4jasUwjrws6Cn6_sPEGmL54sq5R4Z5afUtv-mItTmqZZdxnkRqcJLlg2Y8WbCPogErbsaCDJoABQ7ppaqHetwfM_0yMun6ABOQbIwwl8pspJhpplKwyo700OSpvTT9zlBsu-b35lzXGBRHzv5g_RA

```

在dashboard登录界面，选择令牌，将上面输出的token粘贴到输入框，点击登录就可以已admin登入。

### 常见问题

1.  node节点`kubeadm join`成功，master上看不到

可以查看node节点的hostname是否于集群内其他节点重复，重复的话通过hostnamectl修改

2.  node节点一直处于NotReady状态

查看节点是否安装了flannel，没有需要安装。

通过systemctl status kubelet查看运行日志中的错误信息。

```
 cni.go:213] Unable to update cni config: No networks found in /etc/cni/net.d
3月 31 17:50:00 host-166 kubelet[19833]: E0331 17:50:00.085663   19833 kubelet.go:2170] Container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
```

类似上面的报错，应该是flannel没有安装或者找不到flannel的配置，可以手动将master的`/etc/cni/net.d/10-flannel.conflist`文件手动复制到node节点相应目录下。如果拉取镜像失败，则可以手动拉取。

3.  dashboard卡在登录界面

确保浏览器通过https访问dashboard。

4.  token过期导致kubeadm join时失败


执行kubeadm join时报错如下。

```
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
error execution phase preflight: unable to fetch the kubeadm-config ConfigMap: failed to get config map: Unauthorized
```

这是因为kubeadm生成的token默认24小时过期，在master上执行`kubeadm token create`创建新的token，替换`kubeadm join --token XXX`中的token即可