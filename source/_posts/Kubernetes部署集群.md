---
title: Kubernetes部署集群
date: 2025-11-07 09:59:04
tags: Kubernetes
categories: k8s
top: 32
---

#### 本文档基于操作系统 Centos7/Debian 12/Ubuntu22.04 进行部署 Kubernetes 集群

| 服务器IP | 主机角色 |
| --- | --- |
| 192.168.1.183 | Kubernetes 1（Master、Node） |
| 192.168.1.184 | Kubernetes 2（Master、Node） |
| 192.168.1.185 | Kubernetes 3（Master、Node） |


- 最低节点要求：3 台节点。
- Master 节点支持冗余，一台 Master 宕机，集群仍可正常操作和运行工作负载

##### 要求
- 集群服务器之间网络策略无限制
- 集群服务器之间主机名不能重复
- 主网卡 MAC 地址不能重复【 ip link 查看 】
- product_id 不能重复【 cat /sys/class/dmi/id/product_uuid 】
- kubelet 的6443端口未被占用【nc -vz 127.0.0.1 6443】
- 禁用 swap 内存【 执行 swapoff -a 命令进行禁用，并且 /etc/fstab 中禁用 swap 分区挂载 】

### 配置HOSTS
在 Kubernetes 集群各节点中添加以下 hosts 信息，将 k8s-master 指向三个 master 节点

```
cat >> /etc/hosts << EOF
192.168.1.183 k8s-master
192.168.1.184 k8s-master
192.168.1.185 k8s-master
EOF
```
> 注意 Kubernetes 集群中每个节点都要添加此 hosts 信息，包括集群后续新增的节点。


### 安装Docker容器运行环境

> Kubernetes 集群各节点均需要操作

1、下载docker安装包
```
通过网盘分享的文件：docker-27.3.1.tgz
链接: https://pan.baidu.com/s/1cTTML8CgVsyhQGt-GBUIeQ 提取码: s9av
```
2、安装docker

```
tar -zxvf docker-27.3.1.tgz
mv -f docker/* /usr/local/bin/
```
3、创建docker与containerd目录

```
mkdir /etc/docker
mkdir /etc/containerd
```
4、创建 docker 的 daemon.json 文件

```
cat > /etc/docker/daemon.json <<\EOF
{
"registry-mirrors": ["https://uvlkeb6d.mirror.aliyuncs.com"],
"data-root": "/data/docker",
"max-concurrent-downloads": 10,
"exec-opts": ["native.cgroupdriver=cgroupfs"],
"storage-driver": "overlay2",
"default-address-pools":[{"base":"172.80.0.0/16","size":24}],
"insecure-registries": ["127.0.0.1:5000"]
}
EOF
```

5、创建 containerd 的 config.toml 文件，并修改配置

```
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup =.*/SystemdCgroup = true/g' /etc/containerd/config.toml
sed -i 's#bin_dir =.*#bin_dir = "/usr/local/kubernetes/cni/bin"#' /etc/containerd/config.toml
sed -i 's#sandbox_image =.*#sandbox_image = "127.0.0.1:5000/pause:3.8"#' /etc/containerd/config.toml
sed -i 's#^root =.*#root = "/data/containerd"#' /etc/containerd/config.toml
```
##### 检查 containerd 配置文件

```
grep "SystemdCgroup\|bin_dir\|sandbox_image\|^root =" /etc/containerd/config.toml
```

#### 输出结果如下：

![alt text](/images/k8s/k8s_1.png)

6、配置 docker 的 systemd 文件

```
cat > /etc/systemd/system/docker.service <<EOF
[Unit]
Description=Docker
After=network-online.target
Wants=network-online.target
Requires=containerd.service
[Service]
Type=notify
ExecStart=/usr/local/bin/dockerd --containerd /var/run/containerd/containerd.sock
ExecReload=/bin/kill -s HUP \$MAINPID
LimitNOFILE=1024000
LimitNPROC=infinity
LimitCORE=0
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
[Install]
WantedBy=multi-user.target
EOF
```
7、配置 containerd 的 systemd 文件

```
cat > /etc/systemd/system/containerd.service <<EOF
[Unit]
Description=containerd
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
ExecStart=/usr/local/bin/containerd --config /etc/containerd/config.toml
LimitNOFILE=1024000
LimitNPROC=infinity
LimitCORE=0
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
[Install]
WantedBy=multi-user.target
EOF
```

8、启动 containerd 与 docker 并加入开机自启动

```
systemctl daemon-reload && systemctl restart containerd && systemctl enable containerd
systemctl daemon-reload && systemctl restart docker && systemctl enable docker
```

### 安装CNI插件

> Kubernetes 集群各节点均需要操作

1、下载 cni 插件文件

```
通过网盘分享的文件：cni-plugins-linux-amd64-v1.1.1.tgz
链接: https://pan.baidu.com/s/1Lo7MYvwuLFEIWO9aYZQ8YA 提取码: 6uk3
```

2、创建 cni 文件安装目录

```
mkdir -p /usr/local/kubernetes/cni/bin
```

3、解压 cni 插件到安装目录

```
tar -zxvf cni-plugins-linux-amd64-v1.1.1.tgz -C /usr/local/kubernetes/cni/bin
```

### 安装 K8S 集群所需命令

> 安装 crictl/kubeadm/kubelet/kubectl 命令，Kubernetes 集群各节点均需要操作

1、创建命令安装目录

```
mkdir -p /usr/local/kubernetes/bin
```

2、下载命令文件至安装目录


```
# crictl 文件下载链接，下载完成后上传至目标服务器，然后解压到 /usr/local/kubernetes/bin 目录

通过网盘分享的文件：crictl-v1.25.0-linux-amd64.tar.gz
链接: https://pan.baidu.com/s/1qU_vBLk53Ll6Aza9EKQFTw 提取码: m935

tar -zxvf crictl-v1.25.0-linux-amd64.tar.gz -C /usr/local/kubernetes/bin

# kubeadm 文件下载链接，下载完成后上传至目标服务器 /usr/local/kubernetes/bin/ 目录
通过网盘分享的文件：kubeadm
链接: https://pan.baidu.com/s/1YXEvB25gGjzfbwPX0LVz6g 提取码: ugss

# kubelet 文件下载链接，下载完成后上传至目标服务器 /usr/local/kubernetes/bin/ 目录
通过网盘分享的文件：kubelet
链接: https://pan.baidu.com/s/1BnNKDZ7JiY-bembQzjpBug 提取码: tpr1

# kubectl 文件下载链接，下载完成后上传至目标服务器 /usr/local/kubernetes/bin/ 目录
通过网盘分享的文件：kubectl
链接: https://pan.baidu.com/s/1k7bq8zNZWEcEE_sfKIAxlQ 提取码: 3vpv
```

3、赋予命令文件可执行权限

```
chmod +x /usr/local/kubernetes/bin/*
chown $(whoami):$(groups) /usr/local/kubernetes/bin/*
```

4、配置 systemd 管理 kubelet


```
cat > /etc/systemd/system/kubelet.service <<\EOF
[Unit]
Description=kubelet: The Kubernetes Node Agent
Documentation=https://kubernetes.io/docs/home/
Wants=network-online.target
After=network-online.target

[Service]
ExecStart=/usr/local/kubernetes/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

5、配置 systemd 管理 kubeadm

```
mkdir -p /etc/systemd/system/kubelet.service.d

cat > /etc/systemd/system/kubelet.service.d/10-kubeadm.conf <<\EOF
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
ExecStart=/usr/local/kubernetes/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
EOF
```

6、启动 kubelet 并加入开机自启动

```
systemctl daemon-reload && systemctl restart kubelet && systemctl enable kubelet
```
> 这里 restart 之后无需查看服务状态，后续步骤 kubeadm init 和 kubeadm join 之后该服务会自动拉起

7、配置 K8S 命令所在目录并加入环境变量

##### Centos
```
export PATH=/usr/local/kubernetes/bin/:$PATH
echo 'export PATH=/usr/local/kubernetes/bin/:$PATH' >> /etc/bashrc
```
##### Ubuntu / Debian

```
export PATH=/usr/local/kubernetes/bin/:$PATH
echo 'export PATH=/usr/local/kubernetes/bin/:$PATH' >> /etc/bash.bashrc
```

8、配置防止后续 crictl 拉取镜像出错

```
crictl config runtime-endpoint unix:///run/containerd/containerd.sock
```

#### 安装环境依赖

> Kubernetes 集群各节点均需要操作

1、安装环境依赖 socat/conntrack

###### 服务器支持互联网访问
```
# centos / redhat 使用 yum 安装
yum install -y socat conntrack-tools

# debian / ubuntu 使用 apt 安装
apt install -y socat conntrack
```

###### 不支持互联网访问

```
# socat 文件包下载链接，下载完成上传至目标服务器（此处使用CentOS 7.9，如依赖不匹配需要重新下载）

通过网盘分享的文件：socat-deps-centos7.tar.gz
链接: https://pan.baidu.com/s/1RDoBUJdRL90jY4c5rq_31A 提取码: 28w9

# 解压后进行安装
tar -zxvf socat-deps-centos7.tar.gz
rpm -Uvh --nodeps socat-deps-centos7/*.rpm

# conntrack 文件包下载链接，下载完成上传至目标服务器（此处使用CentOS 7.9，如依赖不匹配需要重新下载）

通过网盘分享的文件：conntrack-tools-deps-centos7.tar.gz
链接: https://pan.baidu.com/s/1fi9WWITsyuT1yMENvKJQag 提取码: 75ej

# 解压后进行安装
tar -zxvf conntrack-tools-deps-centos7.tar.gz
rpm -Uvh --nodeps conntrack-tools-deps-centos7/*.rpm
```


2、检查命令是否缺失

```
docker --version && dockerd --version && pgrep -f 'dockerd' && crictl --version && kubeadm version && kubelet --version && kubectl version --client=true && socat -V | grep 'socat version' && conntrack --version && echo ok || echo error
```

> 输出 ok 代表正常，输出 error 则需根据错误补全命令


#### 修改内核配置
> Kubernetes 集群各节点均需要操作

1、添加内核模块

```
cat > /etc/modules-load.d/kubernetes.conf <<EOF
overlay
br_netfilter
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
EOF
```

2、加载模块

```
modprobe overlay
modprobe br_netfilter
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
```

3、添加内核参数

```
cat >> /etc/sysctl.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
vm.max_map_count = 262144

# MD Config
net.nf_conntrack_max = 524288
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 8192 87380 16777216
net.ipv4.tcp_wmem = 8192 65536 16777216
net.ipv4.tcp_max_syn_backlog = 32768
net.core.netdev_max_backlog =  32768
net.core.netdev_budget = 600
net.core.somaxconn = 32768
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 2
net.ipv4.tcp_mem = 8388608 12582912 16777216
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_max_orphans = 16384
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_time = 600
vm.max_map_count = 262144
net.netfilter.nf_conntrack_tcp_be_liberal = 0
net.netfilter.nf_conntrack_tcp_max_retrans = 3
net.netfilter.nf_conntrack_tcp_timeout_max_retrans = 300
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
fs.inotify.max_user_watches=10485760
fs.inotify.max_user_instances=10240
EOF

sysctl --system
```

#### K8S 环境镜像准备

> Kubernetes 集群各节点均需要操作


1、加载离线镜像
```
通过网盘分享的文件：kubeadm-1.25.4-images.tar.gz
链接: https://pan.baidu.com/s/1iCCeDPJiGg5laToFTfkzSg 提取码: 5een
```

2、启动本地仓库给镜像打标签


```
docker run -d -p 5000:5000 --restart always --name registry registry:2
for i in $(docker images | grep 'registry.k8s.io\|rancher' | awk 'NR!=0{print $1":"$2}');do docker tag $i $(echo $i | sed -e "s/registry.k8s.io/127.0.0.1:5000/" -e "s#coredns/##" -e "s/rancher/127.0.0.1:5000/");done
for i in $(docker images | grep :5000 | awk 'NR!=0{print $1":"$2}');do docker push $i;done
docker images | grep :5000
```

#### 初始化第一个主节点

>仅在 Kubernetes 01 节点操作

1、初始化 master 节点

1.2 命令行方式初始化

```
kubeadm init --control-plane-endpoint "k8s-master:6443" --upload-certs --cri-socket unix:///var/run/containerd/containerd.sock -v 5 --kubernetes-version=1.25.4 --image-repository=127.0.0.1:5000 --pod-network-cidr=10.244.0.0/16
```

1.3 kubeadm-config.yaml方式初始化
生成 kubeadm-config.yaml 配置文件

```
cd /usr/local/kubernetes/
kubeadm config print init-defaults > /usr/local/kubernetes/kubeadm-config.yaml
```
编辑配置文件

```
# 修改镜像
sed -ri 's#imageRepository.*#imageRepository: 127.0.0.1:5000#' /usr/local/kubernetes/kubeadm-config.yaml
# 配置pod网段
sed -ri '/serviceSubnet/a \ \ podSubnet: 10.244.0.0\/16' /usr/local/kubernetes/kubeadm-config.yaml
# 修改节点IP地址
sed -ri 's#advertiseAddress.*#advertiseAddress: '$(hostname -I |awk '{print $1}')'#' /usr/local/kubernetes/kubeadm-config.yaml
# 修改etcd数据目录
sed -ri 's#dataDir:.*#dataDir: /data/etcd#' /usr/local/kubernetes/kubeadm-config.yaml
# 修改节点名称
sed -ri 's#name: node#name: '$(hostname)'#' /usr/local/kubernetes/kubeadm-config.yaml
# 修改kubernetes版本
sed -ri 's#kubernetesVersion.*#kubernetesVersion: 1.25.4#' /usr/local/kubernetes/kubeadm-config.yaml

# 添加 --control-plane-endpoint "k8s-master:6443"
sed -i '/apiServer:/i controlPlaneEndpoint: "k8s-master:6443"' /usr/local/kubernetes/kubeadm-config.yaml

# 查看修改结果
grep 'advertiseAddress\|name\|imageRepository\|dataDir\|podSubnet\|kubernetesVersion\|controlPlaneEndpoint' /usr/local/kubernetes/kubeadm-config.yaml
```
输出结果示例：

```
  advertiseAddress: 192.168.1.183
  name: service01
controlPlaneEndpoint: "k8s-master:6443"
    dataDir: /data/etcd
imageRepository: 127.0.0.1:5000
kubernetesVersion: 1.25.4
  podSubnet: 10.244.0.0/16
```

kubeadm-config.yaml示例


#### 注意修改 advertiseAddress ip地址
```
apiVersion: kubeadm.k8s.io/v1beta3
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
  advertiseAddress: 192.168.1.183 # master ip
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: service01 # master hostname
  taints: null
---
controlPlaneEndpoint: "k8s-master:6443"
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /data/etcd
imageRepository: 127.0.0.1:5000
kind: ClusterConfiguration
kubernetesVersion: 1.25.4
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
scheduler: {}
```

初始化master节点

```
# 查看所需镜像列表
kubeadm config images list --config /usr/local/kubernetes/kubeadm-config.yaml
# 拉取镜像
kubeadm config images pull --config /usr/local/kubernetes/kubeadm-config.yaml
# 检查
kubeadm init phase preflight --config=/usr/local/kubernetes/kubeadm-config.yaml
# 根据配置文件启动 kubeadm 初始化 k8s
kubeadm init --config=/usr/local/kubernetes/kubeadm-config.yaml --upload-certs --v=6
```

---


1.4 尾部输出类似于：

```
...
You can now join any number of control-plane node by running the following command on each as a root:
    kubeadm join k8s-master:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:
    kubeadm join k8s-master:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866
```

将此输出复制到文本文件。 稍后你将需要它来将 master 和 node 节点加入集群。


2、修改 nodePort 可使用端口范围

```
sed -i '/- kube-apiserver/a\ \ \ \ - --service-node-port-range=1024-32767' /etc/kubernetes/manifests/kube-apiserver.yaml
```

3、设置配置路径

Centos
```
export KUBECONFIG=/etc/kubernetes/admin.conf
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> /etc/bashrc
```

Debian/Ubuntu
```
export KUBECONFIG=/etc/kubernetes/admin.conf
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> /etc/bash.bashrc
```

4、调整当前节点 Pod 上限

```
echo "maxPods: 300" >> /var/lib/kubelet/config.yaml
systemctl restart kubelet
```

5、允许 master 参与调度


- 在初始化 master 节点后大概要等待1-2分钟左右再执行下方命令
- 执行前需先检查 kubelet 服务状态 systemctl status kubelet，看下是否为 running

```
kubectl taint node $(kubectl get node | grep control-plane | awk '{print $1}') node-role.kubernetes.io/control-plane:NoSchedule-
```

> 此命令执行后，正确输出为："xxxx untainted"，如果输出不符，则需稍加等待，再次执行进行确认


6、安装网络插件

```
cat > /usr/local/kubernetes/kube-flannel.yml <<EOF
---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: flannel
  namespace: kube-system
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-system
  labels:
    tier: node
    app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
       #image: flannelcni/flannel-cni-plugin:v1.1.0 for ppc64le and mips64le (dockerhub limitations may apply)
        image: 127.0.0.1:5000/mirrored-flannelcni-flannel-cni-plugin:v1.1.0
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
       #image: flannelcni/flannel:v0.20.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: 127.0.0.1:5000/mirrored-flannelcni-flannel:v0.20.1
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
       #image: flannelcni/flannel:v0.20.1 for ppc64le and mips64le (dockerhub limitations may apply)
        image: 127.0.0.1:5000/mirrored-flannelcni-flannel:v0.20.1
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /usr/local/kubernetes/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
EOF

kubectl apply -f /usr/local/kubernetes/kube-flannel.yml
```

#### 将其他主节点加入集群

> 需在Kubernetes 02/03 节点上进行操作

1、加入 Kubernetes 集群

```
kubeadm join 192.168.1.183:6443 --token 58kbiu.fqueeaddjwo6cemv --discovery-token-ca-cert-hash sha256:9490f7a7961fa6277c8f8807ae14c28ef997f332732af90c02ed029214488821
```

- 此命令为在主节点执行 kubeadm init 成功后输出，此处的为示例，每个集群都不同
- 如遗忘的话请参考以下步骤在第一个主节点重新获取：
- 重新生成 join 命令

```
kubeadm token create --print-join-command
```
- 重新上传证书并生成新的解密密钥

```
kubeadm init phase upload-certs --upload-certs
```
- 拼接 join 命令，新增 --control-plane --certificate-key 参数，并将生成的解密密钥作为 --certificate-key 参数值

```
kubeadm join k8s-master:6443 --token 1b6i9d.0qqufwsjrjpuhkwo --discovery-token-ca-cert-hash sha256:3d28faa49e9cac7dd96aded0bef33a6af1ced57e45f0b12c6190f3d4e1055456 --control-plane --certificate-key 57a0f0e9be1d9f1c74bab54a52faa143ee9fd9c26a60f1b3b816b17b93ecaf6f
```
至此，得到了 master 节点加入集群的 join 命令


2、修改 nodePort 可使用端口范围

```
sed -i '/- kube-apiserver/a\ \ \ \ - --service-node-port-range=1024-32767' /etc/kubernetes/manifests/kube-apiserver.yaml
```

3、设置配置路径

Centos
```
export KUBECONFIG=/etc/kubernetes/admin.conf
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> /etc/bashrc
```

Debian/Ubuntu
```
export KUBECONFIG=/etc/kubernetes/admin.conf
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> /etc/bash.bashrc
```

4、调整当前节点 Pod 上限

```
echo "maxPods: 300" >> /var/lib/kubelet/config.yaml
systemctl restart kubelet
```

5、允许 master 参与调度
- 在初始化完当前节点后大概要等待1-2分钟左右再执行下方命令
- 执行前需先检查 kubelet 服务状态 systemctl status kubelet，看下是否为 running
```
kubectl taint node $(kubectl get node | grep control-plane | awk '{print $1}') node-role.kubernetes.io/control-plane:NoSchedule-
```
- 此命令执行后，正确输出为："xxxx untainted"，如果输出不符，则需稍加等待，再次执行进行确认



#### 新增工作节点加入集群（如有）

> 例如 api/db/file 节点或后续继续新增的微服务节点，都是以工作节点加入当前多 master 的 kubernetes 集群

1、加入 kubernetes 集群

```
kubeadm join 192.168.1.183:6443 --token 58kbiu.fqueeaddjwo6cemv --discovery-token-ca-cert-hash sha256:9490f7a7961fa6277c8f8807ae14c28ef997f332732af90c02ed029214488821
```

- 此命令为在主节点执行 kubeadm init 成功后输出，此处的为示例，每个集群都不同
- 如遗忘的话可以在主节点执行 ku beadm token create --print-join-command 重新获取


2、调整当前节点 Pod 上限

```
echo "maxPods: 300" >> /var/lib/kubelet/config.yaml
systemctl restart kubelet
```

#### 集群状态检查

```
kubectl get pod -n kube-system    # READY列需要是"1/1"
kubectl get node                  # STATUS列需要是"Ready"
```

#### 最终检查如下

![alt text](/images/k8s/k8s_res.png)