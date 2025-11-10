---
date: 2020-08-23
tags:  [k8s,docker]
---

# 搭建k8s集群

引用 https://blog.piaoruiqing.com/2019/09/17/kubernetes-1-installation/
照着文章搭建k8s集群，写得挺好，记录一下自己的搭建方法和问题

## 环境 

1. 2个kvm虚拟机, 安装的centos7系统
2. 两个虚拟机都配置了双网卡，eth0和eth1， eth0桥接宿主机的bridge， eth1 接入libvirt default NAT网络
3. 官方推荐配置： CPU > 2，内存 > 2G
4. eth1 的ip 分别为 192.168.122.61, 192.168.122.161

## Master和Worker 共同步骤

需要在所有节点执行

### 修改hostname

编辑hostname
```
vim /etc/hostname # 修改hostname
```
配置host文件， 将所有节点都指定host 

### 关闭防火墙

1. systemctl disable --now firewalld
2. 修改 /etc/seconfig， 将selinux 配置成disable 

### 关闭 swap 分区

1. swapoff -a 
2. 修改 /etc/fstab， 注释掉swap 记录

### 安装docker 配置镜像源

修改 /etc/docker/daemon.json 文件
 
```
{
  "registry-mirrors": ["https://xxxxxxxx.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```
  
 ### 添加k8s 软件源 
 
 如下添加的aliyun的镜像源
 
 ```
 cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
```

### 安装k8s

安装组件 

```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable kubelet && systemctl start kubelet
``` 

### 配置网络


```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

上面的步骤需要在主备上执行

## Master 操作 

下面的步骤只在Master节点运行

### 生成k8s初始化配置文件 

```
[root@k8s-master ~]$ kubeadm config print init-defaults > kubeadm-init.yaml
```
得到 kubeadm-init.yaml 文件

修改kubeadm-init.yaml 文件
1. 修改 `advertiseAddress: 1.2.3.4` 为本机地址 192.168.122.61
2. 修改 `imageRepository: k8s.gcr.io ` 为 imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers 

修改镜像地址，这样就避免下载k8s 镜像超时

修改后的kubeadm-init.yaml

```yaml
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.33.30.92
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.15.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

### 下载k8s的组件镜像 

```
kubeadm config images pull --config kubeadm-init.yaml
```

### 初始化 k8s

```
kubeadm init --config kubeadm-init.yaml
```
执行无误后，得到worker 节点加入集群的命令 
```
...
Your Kubernetes control-plane has initialized successfully!
...
Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.33.30.92:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:2883b1961db36593fb67ab5cd024f451b934fc0e72e2fa3858dda3ad3b225837 
```

记录这个命令， 丢失的话也可以重新执行 `kubeadmin create token ` 找回

### 配置普通用户执行kubectl 命令 

用普通用户执行一下命令， 这样普通用户也能管理集群 
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 查看Master 节点状态 

可见，Master节点未就绪，这是因为还没安装网络组件
```
[root@k8s-master kubernetes]$ kubectl get node
NAME         STATUS     ROLES    AGE     VERSION
k8s-master   NotReady   master   3m25s   v1.15.3
```


### 配置网络 

1. 下载calico 描述文件 
wget https://docs.projectcalico.org/v3.8/manifests/calico.yaml

2. 修改calico 描述文件里的serviceSubnet，修改为跟kubeadmin-init.yaml一致
```
cat kubeadm-init.yaml | grep serviceSubnet:
serviceSubnet: 10.96.0.0/12
```

***注意***   
1. calico 的地址必须和 kubeadm-init.yaml 保持一致， kubeadm-init.yaml 默认为 `serviceSubnet: 10.96.0.0/12`， 而calico的默认地址为`192.168.0.0/16`, 所以要么修改kubeadmin-init.yaml, 要么修改calico.yaml。
2. 这里的`Subnet` 不能涵盖到主机的地址范围即不能包含 `192.168.122.0/24` 

### 安装calico 

```
kubectl apply -f calico.yaml 
```

### 查看Master节点 

当calico 安装好了， 可以发现Master 节点变为就绪， 至此Master 节点就配置好了
```
[root@k8s-master ~]$ kubectl get node
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   15m   v1.15.3
```

## 安装dashboard 

之前的步骤安装好了Master节点， 可选择安装dashboard，通过网页来管理集群 

步骤很简单， 先下载dashboard， 再安装 
```
[root@k8s-master ~]$ wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
[root@k8s-master ~]$ kubectl apply -f recommended.yaml 
```

### 创建dashboard 用户 

必须创建用户，然后获取token，才能在集群外访问， 否则必须在Master 的localhost 访问dashboard。 
以下是创建用户的描述文件， 注意namespace， 例如官方的例子是 dashboard， 这样生成的token 不一样

执行安装用户 `kubectl apply -f dashboard-adminuser.yaml`

```
# dashboard-adminuser.yaml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
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

创建证书， k8s 不允许远程使用http，所以需要用证书访问dashboard 

```
[root@k8s-master ~]$ grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
[root@k8s-master ~]$ grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
[root@k8s-master ~]$ openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```

将生成的 `kubecfg.p12` 文件导入到集群外的主机，即kvm宿主机上

```
scp root@10.33.30.92:/root/.kube/kubecfg.p12 ./

```

在宿主机上使用chrome 的高级功能里可以导入证书  

重启chrome 登录后登录
```
https://192.168.122.61:6443/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/login
```

打开网页后选择输入token， token通过在Master 节点执行一下命令生成， 注意参数 -n kube-system， 需要跟创建用户时的namespace保持一致 
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

复制token 到网页上，就能登录Dashboard 

## Worker节点加入集群 

直接在worker节点执行 

```
kubeadm join 10.33.30.92:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:2883b1961db36593fb67ab5cd024f451b934fc0e72e2fa3858dda3ad3b225837
```

然后到Master 节点执行以下命令， 这里Name 显示了k8s-worker是因为我们在Hosts文件里指定了ip，所以k8s自动识别到了，否则会显示节点ip

```
[root@k8s-master ~]$ kubectl get node
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    master   10h   v1.15.3
k8s-worker   Ready    <none>   96s   v1.15.3

```

至此集群搭建完毕

## 遇到问题

1. 因为我的虚拟机配置的双网卡， 需要在calico.yaml 配网卡interface
```
# Cluster type to identify the deployment type
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
            - name: IP_AUTODETECTION_METHOD
              value: "interface=eth1"

```
