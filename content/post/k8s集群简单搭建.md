+++
title = 'K8s集群简单搭建'
date = 2023-10-05T12:32:30+08:00
+++

# K8S简单集群搭建

## 前提条件

- windos11电脑，内存16g以上
- 安装vmware虚拟机软件
- 安装三个centos7虚拟机，分配硬盘40g,内存4g,CPU4核心
- 网络均采用NAT模式（新建虚拟机默认的模式）

centos7镜像下载：https://mirrors.tuna.tsinghua.edu.cn/centos/7/isos/x86_64/CentOS-7-x86_64-Everything-2207-02.iso

我电脑上三个centos7虚拟机均采用最小化安装，IP如下：

| 名称          | IP地址           |
| ----------- | -------------- |
| k8s-master1 | 192.169.94.132 |
| k8s-node1   | 192.168.94.133 |
| k8s-node2   | 192.168.94.134 |

其中硬盘分配：

- /boot 1024M
- swap 2048M
- /  37G

## 系统准备

如下命令，没有特殊说明，则在三个节点上都要执行一次

### 关闭防火墙

```bash
systemctl stop firewalld
systemctl disable firewalld
```

### 关闭selinux

```bash
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时
```

### 关闭swap

```bash
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久
```

### 设置主机名

- 在master虚拟机上执行

  ```bash
  hostnamectl set-hostname k8s-master1
  ```

- 在node1虚拟机上执行

  ```bash
  hostnamectl set-hostname k8s-node1
  ```

- 在node2虚拟机上执行

  ```bash
  hostnamectl set-hostname k8s-node2
  ```

### 添加hosts

在master虚拟机上执行

```bash
cat >> /etc/hosts << EOF
192.168.94.132 k8s-master1
192.168.94.133 k8s-node1
192.168.94.134 k8s-node2
EOF
```

### 将桥接的IPv4流量传递到iptables的链

````bash
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
````

生效：

```bash
sysctl --system
```

### 时间同步

```bash
yum install ntpdate -y
ntpdate time.windows.com
```

### 可能的问题

- 无法使用`ifconfig` 命令

  需要安装net-tools:

  ```bash
   yum install net-tools
  ```

  ​

## k8s安装

k8s的默认容器运行时（CRI）为docker,因此要先安装docker

### docker安装

```bash
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

yum -y install docker-ce-18.06.1.ce-3.el7

systemctl enable docker && systemctl start docker
```

验证docker是否安装好：

```bash
docker --version
```

如果下方出现了docker的版本，则说明安装没问题。

设置docker的镜像拉取地址：

```bash
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

### kubeadm/kubelet/kubectl的安装

#### 设置k8s的软件源

```bash
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg 
EOF
```

#### 安装

注意：最好是指定版本，因为坑已踩过

```
$ yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0
$ systemctl enable kubelet
```

#### 初始化Master

在master1上执行集群的master节点初始化：(指定阿里云镜像仓库)

```bash
kubeadm init \
  --apiserver-advertise-address=192.168.94.132 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.18.0 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16
```

执行这句话之后，控制台输出如下：

```bash
[root@localhost ~]# kubeadm init \
>   --apiserver-advertise-address=192.168.94.132 \
>   --image-repository registry.aliyuncs.com/google_containers \
>   --kubernetes-version v1.18.0 \
>   --service-cidr=10.96.0.0/12 \
>   --pod-network-cidr=10.244.0.0/16
W1004 11:44:59.317714   70492 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.0
[preflight] Running pre-flight checks
        [WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.94.132]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master1 localhost] and IPs [192.168.94.132 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master1 localhost] and IPs [192.168.94.132 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W1004 11:45:44.142408   70492 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W1004 11:45:44.143008   70492 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 14.503883 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node k8s-master1 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-master1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: t6p6ko.ok8x7h1era4pq66e
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.94.132:6443 --token t6p6ko.ok8x7h1era4pq66e \
    --discovery-token-ca-cert-hash sha256:c1b3fd084dac33494a27532c368c844181e6942c5c0d2007c2e683ac3ffea83a 

```

注意最后这一句,这是初始化master几点生成的node加入时使用的token,默认为24小时,后面会用到

```bash
kubeadm join 192.168.94.132:6443 --token t6p6ko.ok8x7h1era4pq66e \
    --discovery-token-ca-cert-hash sha256:c1b3fd084dac33494a27532c368c844181e6942c5c0d2007c2e683ac3ffea83a
```

当token过期后，想其他的node节点加入，可通过如下命令重新创建token:

```bash
kubeadm token create --print-join-command
```

#### 部署CNI网络插件

在master节点上执行（下载yml文件）：

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

在master节点上应用文件：

```bash
kubectl apply -f kube-fannel.yml
```

可能出现问题：拉取yml文件失败：

```bash
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
--2023-10-04 11:50:46--  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
正在解析主机 raw.githubusercontent.com (raw.githubusercontent.com)... ::1, 127.0.0.1
正在连接 raw.githubusercontent.com (raw.githubusercontent.com)|::1|:443... 失败：拒绝连接。
正在连接 raw.githubusercontent.com (raw.githubusercontent.com)|127.0.0.1|:443... 失败：拒绝连接。
```

解决办法：参考[wget安装flannel插件-连接失败_Geray-zsg的博客-CSDN博客](https://blog.csdn.net/Entity_G/article/details/113759809)

在`/etc/hosts` 文件中添加如下解析：

```
199.232.68.133 raw.githubusercontent.com
```

再执行就可以正常下载了

## k8s安装后测试

在k8s集群中创建一个pod:

```bash
kubectl create deployment nginx --image=nginx
```

暴露端口：

```bash
kubectl expose deployment nginx --port=80 --type=NodePort
```

查看暴露的服务：

```bash
[root@localhost etc]# kubectl get pod,svc
NAME                        READY   STATUS    RESTARTS   AGE
pod/nginx-f89759699-hnhl9   1/1     Running   0          59m

NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        67m
service/nginx        NodePort    10.96.178.17   <none>        80:32377/TCP   46m

```

验证访问：

http://192.168.94.132:32377

http://192.168.94.133:32377

http://192.168.94.134:32377

如果都能访问，代表k8s集群部署正常



