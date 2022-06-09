# k8s v.1.21集群安装部署过程（kubeadm方式）

## 一、设置主机名

``` text
hostnamectl set-hostname k8s-master
hostnamectl set-hostname k8s-node01
hostnamectl set-hostname k8s-node01
```

## 二、升级内核

1.安装内核

``` text
wget https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/k8s-install/kernel-lt-5.4.114-1.el7.elrepo.x86_64.rpm
wget https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/k8s-install/kernel-lt-devel-5.4.114-1.el7.elrepo.x86_64.rpm
yum -y install  kernel-lt-5.4.114-1.el7.elrepo.x86_64.rpm kernel-lt-devel-5.4.114-1.el7.elrepo.x86_64.rpm
```

2.调整默认内核启动

``` text
grub2-set-default "CentOS Linux (5.4.114-1.el7.elrepo.x86_64) 7 (Core)"
```

3.检查是否修改正确并重启系统

``` text
grub2-editenv list
reboot
```

## 三、将桥接的IPv4流量传递到iptables的链

``` text
modprobe br_netfilter
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system 
```

## 四、开启IPVS支持

创建/etc/sysconfig/modules/ipvs.modules文件，内容如下：

``` text
#!/bin/bash
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_fo ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack"
for kernel_module in ${ipvs_modules}; do
  /sbin/modinfo -F filename ${kernel_module} > /dev/null 2>&1
  if [ $? -eq 0 ]; then
    /sbin/modprobe ${kernel_module}
  fi
done
```

最后，执行如下命令使配置生效：

``` text
chmod 755 /etc/sysconfig/modules/ipvs.modules 
sh /etc/sysconfig/modules/ipvs.modules
lsmod | grep ip_vs
```

下载ipvs管理工具

``` text
yum install -y ipset ipvsadm
```

## 五、同步服务器时间

``` text
yum install chrony -y
systemctl enable chronyd --now
chronyc sources
```

## 六、关闭防火墙、selinux

1.K8s集群每个节点都需要关闭防火墙，执行如下操作：

``` text
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

2.接着，还需要关闭系统的交换分区，执行如下命令：

``` text
swapoff -a
cp /etc/fstab  /etc/fstab.bak
cat /etc/fstab.bak | grep -v swap > /etc/fstab
```

3.然后，还需要修改iptables设置，在/etc/sysctl.conf中添加如下内容：

``` text
cat >> /etc/sysctl.conf << EOF
vm.swappiness = 0
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

4.最后，执行如下命令，以使设置生效：

``` text
sysctl -p
```

## 七、主机名本地解析配置
每个主机的主机名以及IP地址都在上面环境介绍中给出来了，根据这些信息，在每个k8s集群节点添加如下主机名解析信息，将这些信息添加到每个集群节点的/etc/hosts文件中，主机名解析内容如下：

``` text
127.0.1.1    i-jg8xmnmc
192.168.0.21 k8s-master
192.168.0.22 k8s-node01
192.168.0.23 k8s-node02
```

注意：使用内网地址

## 八、安装docker环境（所有节点都执行下面步骤）

1.安装

``` text
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum install docker-ce -y
systemctl restart docker
systemctl enable docker
docker version
```

2.配置

``` text
cat <<EOF | tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
```

3.重新启动 Docker 并在启动时启用：

``` text
systemctl enable docker
systemctl daemon-reload
systemctl restart docker
```

## 九、安装Containerd

1.下载并配置

``` text
cd /root/
yum install libseccomp -y

wget https://linktime-external-project.oss-cn-hangzhou.aliyuncs.com/k8s-install/cri-containerd-cni-1.5.5-linux-amd64.tar.gz
tar -C / -xzf cri-containerd-cni-1.5.5-linux-amd64.tar.gz

echo "export PATH=$PATH:/usr/local/bin:/usr/local/sbin" >> ~/.bashrc
source ~/.bashrc

mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

systemctl enable containerd --now
ctr version
```

2.将 containerd 的 cgroup driver 配置为 systemd

``` text
#通过搜索SystemdCgroup进行定位
vim /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
    ....

#注意：最终输出shell命令：
sed -i "s/SystemdCgroup = false/SystemdCgroup = true/g" /etc/containerd/config.toml
```

3.配置镜像加速器地址

然后再为镜像仓库配置一个加速器，需要在 cri 配置块下面的 registry 配置块下面进行配置 registry.mirrors：（注意缩进）
![image](https://note.youdao.com/yws/res/1879/25D274F0303847D18822B016B817B8BE)

``` text
[plugins."io.containerd.grpc.v1.cri".registry.mirrors]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
    endpoint = ["https://kvuwuws2.mirror.aliyuncs.com"]
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
    endpoint = ["https://registry.aliyuncs.com/k8sxio"]
```

4.启动containerd服务

``` text
systemctl daemon-reload
systemctl enable containerd --now
```

5.验证

``` text
ctr version
crictl version
```

## 十、安装Kubeadm、kubelet、kubectl工具

1.在确保系统基础环境配置完成后，现在我们就可以来安装 Kubeadm、kubelet了，我这里是通过指定yum 源的方式来进行安装，使用阿里云的源进行安装：

``` text
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

2.安装 kubeadm、kubelet、kubectl(all节点均要配置)。

``` text
yum makecache fast
yum install -y kubelet-1.22.2 kubeadm-1.22.2 kubectl-1.22.2 --disableexcludes=kubernetes
kubeadm version

systemctl enable --now kubelet
kubeadm version
```

## 十一、初始化集群(master节点操作)

1.执行如下命令:

``` text
kubeadm config print init-defaults
kubeadm config images list
kubeadm config images list --kubernetes-version=v1.21.0 --image-repository swr.myhuaweicloud.com/iivey
```

其中：
第一个命令用来查看安装k8s的相关信息，主要是安装源和版本。
第二条命令是查询需要的镜像，从输出可在，默认是从k8s.gcr.io这个地址下载镜像，此地址国内无法访问，因此需要修改默认下载镜像的地址。
第三条命令是设置k8s镜像仓库为华为云镜像站地址，查看一下需要下载的镜像都有哪些。

2.接着，在 master 节点配置 kubeadm 初始化文件，可以通过如下命令导出默认的初始化配置：

``` text
kubeadm config print init-defaults --component-configs KubeletConfiguration > kubeadm.yaml
```

3.然后根据我们自己的需求修改配置，比如修改 imageRepository 指定集群初始化时拉取 Kubernetes 所需镜像的地址，kube-proxy 的模式为 ipvs，另外需要注意的是我们这里是准备安装 flannel 网络插件的，需要将 networking.podSubnet 设置为10.244.0.0/16。

``` text
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
  advertiseAddress: 192.168.0.21  # 修改1：指定master节点内网IP
  bindPort: 6443
nodeRegistration:
  criSocket: /run/containerd/containerd.sock  # 修改2：使用 containerd的Unix socket 地址
  imagePullPolicy: IfNotPresent
  name: k8s-master01 #name #修改3：修改master节点名称
  taints:  # 修改4：给master添加污点，master节点不能调度应用
  - effect: "NoSchedule"
    key: "node-role.kubernetes.io/master"

---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs  # 修改5：修改kube-proxy 模式为ipvs，默认为iptables

---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/k8sxio #修改6：image地址
kind: ClusterConfiguration
kubernetesVersion: 1.22.2 #修改7：指定k8s版本号，默认这里忽略了小版本号
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16  # 修改8：指定 pod 子网
scheduler: {}

---
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
clusterDNS:
- 10.96.0.10
clusterDomain: cluster.local
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
cgroupDriver: systemd  # 修改9：配置 cgroup driver
logging: {}
memorySwap: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
```

4.提前下载镜像

``` text
kubeadm config images list
kubeadm config images list --config kubeadm.yaml
kubeadm config images pull --config kubeadm.yaml
```

5.上面在拉取 coredns 镜像的时候出错了，阿里云仓库里没有找到这个镜像，我们可以手动到官方仓库 pull 该镜像，然后重新 tag 下镜像地址即可

``` text
ctr -n k8s.io i pull docker.io/coredns/coredns:1.8.4

ctr -n k8s.io i tag docker.io/coredns/coredns:1.8.4 registry.aliyuncs.com/k8sxio/coredns:v1.8.4
```

6.pause镜像会使用k8s.gcr.io镜像仓库，提前打上tag

``` text
ctr -n k8s.io i tag registry.aliyuncs.com/k8sxio/pause:3.5 k8s.gcr.io/pause:3.5
ctr -n k8s.io i ls -q
```

或者修改/etc/containerd/config.toml文件，sandbox_image字段

7.初始化master节点

``` text
kubeadm init --config kubeadm.yaml
```

8.根据安装提示拷贝 kubeconfig 文件

``` text
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

9.查看安装效果

``` text
kubectl get node
```

## 十二、添加节点

``` text
kubeadm join 192.168.0.21:6443 --token abcdef.0123456789abcdef \
--discovery-token-ca-cert-hash sha256:ad45d9d60ddab962d72c054de05f41e130b4cf42a75d8e97e96b63331cf9fbf1
```

join 命令：如果忘记了上面的 join 命令可以使用命令 kubeadm token create --print-join-command 重新获取。

## 十三、安装网络插件flannel

这个时候其实集群还不能正常使用，因为还没有安装网络插件，接下来安装网络插件，可以在文档 <https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/> 中选择我们自己的网络插件，这里我们安装 flannel:

``` text
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

这个文件已经配置好，直接apply:

``` text
kubectl apply -f kube-flannel.yml
```

隔一会儿查看 Pod 运行状态:

``` text
kubectl get pod -nkube-system -owide
```

## 十四、Dashboard

1.下载kube-dashboard的yaml文件

``` text
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
```

2.文件内容

``` text
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

apiVersion: v1
kind: Namespace
metadata:
  name: kubernetes-dashboard

---

apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kubernetes-dashboard
type: Opaque

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-csrf
  namespace: kubernetes-dashboard
type: Opaque
data:
  csrf: ""

---

apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-key-holder
  namespace: kubernetes-dashboard
type: Opaque

---

kind: ConfigMap
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-settings
  namespace: kubernetes-dashboard

---

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
rules:
  # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]

---

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kubernetes-dashboard
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kubernetes-dashboard

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
    spec:
      containers:
        - name: kubernetes-dashboard
          image: kubernetesui/dashboard:v2.3.1
          imagePullPolicy: Always
          ports:
            - containerPort: 8443
              protocol: TCP
          args:
            - --auto-generate-certificates
            - --namespace=kubernetes-dashboard
            # Uncomment the following line to manually specify Kubernetes API server Host
            # If not specified, Dashboard will attempt to auto discover the API server and connect
            # to it. Uncomment only if the default does not work.
            # - --apiserver-host=http://my-address:port
          volumeMounts:
            - name: kubernetes-dashboard-certs
              mountPath: /certs
              # Create on-disk volume to store exec logs
            - mountPath: /tmp
              name: tmp-volume
          livenessProbe:
            httpGet:
              scheme: HTTPS
              path: /
              port: 8443
            initialDelaySeconds: 30
            timeoutSeconds: 30
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      volumes:
        - name: kubernetes-dashboard-certs
          secret:
            secretName: kubernetes-dashboard-certs
        - name: tmp-volume
          emptyDir: {}
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule

---

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 8000
      targetPort: 8000
  selector:
    k8s-app: dashboard-metrics-scraper

---

kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    k8s-app: dashboard-metrics-scraper
  name: dashboard-metrics-scraper
  namespace: kubernetes-dashboard
spec:
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: dashboard-metrics-scraper
  template:
    metadata:
      labels:
        k8s-app: dashboard-metrics-scraper
      annotations:
        seccomp.security.alpha.kubernetes.io/pod: 'runtime/default'
    spec:
      containers:
        - name: dashboard-metrics-scraper
          image: kubernetesui/metrics-scraper:v1.0.6
          ports:
            - containerPort: 8000
              protocol: TCP
          livenessProbe:
            httpGet:
              scheme: HTTP
              path: /
              port: 8000
            initialDelaySeconds: 30
            timeoutSeconds: 30
          volumeMounts:
          - mountPath: /tmp
            name: tmp-volume
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsUser: 1001
            runAsGroup: 2001
      serviceAccountName: kubernetes-dashboard
      nodeSelector:
        "kubernetes.io/os": linux
      # Comment the following tolerations if Dashboard must not be deployed on master
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      volumes:
        - name: tmp-volume
          emptyDir: {}
```

3.修改kube-dashboard.yaml文件

``` text
# 修改Service为NodePort类型
......
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort  # 加上type=NodePort变成NodePort类型的服务
......
```

4.部署kube-dashboard.yaml文件

``` text
mv recommended.yaml kube-dashboard.yaml
kubectl apply -f kube-dashboard.yaml
```

## 十五、配置cni

我们仔细看可以发现上面的 Pod 分配的 IP 段是 10.88.xx.xx，包括前面自动安装的 CoreDNS 也是如此，我们前面不是配置的 podSubnet 为 10.244.0.0/16 吗？

``` text
kubectl get pod -A -owide
```

![image](https://note.youdao.com/yws/res/1947/081E9056DBCC4BE588E6ED1CF17D3C6C)

我们先去查看下 CNI 的配置文件

``` text
ll /etc/cni/net.d/
```

![image](https://note.youdao.com/yws/res/1952/61CE65A8EC1A4B128516732CDA5409E4)

可以看到里面包含两个配置，一个是 10-containerd-net.conflist，另外一个是我们上面创建的 Flannel 网络插件生成的配置，我们的需求肯定是想使用 Flannel 的这个配置，我们可以查看下 containerd 这个自带的 cni 插件配置：

``` text
cat /etc/cni/net.d/10-containerd-net.conflist
{
  "cniVersion": "0.4.0",
  "name": "containerd-net",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "promiscMode": true,
      "ipam": {
        "type": "host-local",
        "ranges": [
          [{
            "subnet": "10.88.0.0/16"
          }],
          [{
            "subnet": "2001:4860:4860::/64"
          }]
        ],
        "routes": [
          { "dst": "0.0.0.0/0" },
          { "dst": "::/0" }
        ]
      }
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true}
    }
  ]
}
```

可以看到上面的 IP 段恰好就是 10.88.0.0/16，但是这个 cni 插件类型是 bridge 网络，网桥的名称为 cni0：
![image](https://note.youdao.com/yws/res/1958/0FEE0A4A3FCB4232A8150D53AB141CE3)

但是使用 bridge 网络的容器无法跨多个宿主机进行通信，跨主机通信需要借助其他的 cni 插件，比如上面我们安装的 Flannel，或者 Calico 等等，由于我们这里有两个 cni 配置，所以我们需要将 10-containerd-net.conflist 这个配置删除，因为如果这个目录中有多个 cni 配置文件，kubelet 将会使用按文件名的字典顺序排列的第一个作为配置文件，所以前面默认选择使用的是 containerd-net 这个插件。

``` text
mv /etc/cni/net.d/10-containerd-net.conflist{,.bak}
ifconfig  cni0 down && ip link delete cni0
systemctl daemon-reload
systemctl restart containerd kubelet
```

然后记得重建 coredns 和 dashboard 的 Pod，重建后 Pod 的 IP 地址就正常了：

``` text
kubectl delete pod 
```

## 十六、创建一个具有全局所有权限的用户来登录 Dashboard：(admin.yaml)

``` text
vim admin.yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kubernetes-dashboard
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kubernetes-dashboard
```

直接创建

``` text
kubectl apply -f admin.yaml
```

``` text
kubectl get secret admin-token-xxxxx -o jsonpath={.data.token} -n kubernetes-dashboard |base64 -d
```

![image](https://note.youdao.com/yws/res/1977/5447B3E422DD40D9BC9D8CD8D6563C5C)
然后用上面的 base64 解码后的字符串作为 token 登录 Dashboard 即可

![image](https://note.youdao.com/yws/res/1979/6CFA93809F294447BDFFC90CA79CEE64)

## 十七、修改node节点container-runtime

执行kubectl get nodes -o wide命令，可以看的node01,node02还是使用docker作为运行时容器
![image](https://note.youdao.com/yws/res/1989/24D83400FB13489790817ECE85501A94)

1.维护节点
首先将需要修改的节点设置成不可调度

``` text
kubectl cordon k8s-node02
kubectl get nodes -o wide
```

驱逐该节点上除了daemonset的pod

``` text
kubectl drain k8s-node02 --ignore-daemonsets
```

2.切换Runtime
关闭docker、containerd 和 kubelet：

``` text
systemctl stop kubelet
systemctl stop docker
systemctl stop containerd
```

3.安装containerd
我们在使用docker-ce作为集群runtime时默认安装了containerd，先将其卸载。

``` text
yum remove docker-ce docker-ce-cli containerd.io
```

安装containerd可参考上面操作

/etc/containerd/config.toml 修改默认的 pause 镜像为国内的地址，替换 [plugins."io.containerd.grpc.v1.cri"] 下面的 sandbox_image

配置下镜像仓库的加速器地址

4.配置kubelet
修改 kubelet 配置，将容器运行时配置为 containerd，编辑/etc/sysconfig/kubelet 文件，在该文件中可以添加kubelet 启动参数：

``` text
KUBELET_EXTRA_ARGS="--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
```

--container-runtime ：指定使用的容器运行时的，可选值为 docker 或者 remote，默认是 docker，除 docker 之外的容器运行时都应该指定为 remote。
--container-runtime-endpoint：是用来指定远程的运行时服务的 endpiont 地址的，在 Linux 系统中一般都是使用 unix 套接字的形式，unix:///run/containerd/containerd.sock。
--image-service-endpoint：指定远程 CRI 的镜像服务地址，如果没有指定则默认使用 --container-runtime-endpoint 的值了，因为 CRI 都会实现容器和镜像服务的。

5.配置完成后重启 containerd 和 kubelet 即可

``` text
systemctl daemon-reload
systemctl restart containerd
systemctl restart kubelet
```

6.查看服务

``` text
crictl -v
kubectl get nodes -o wide
```

7.运行调度

```text
kubectl uncordon k8s-node02
```

## 十八、参考地址

<https://mdnice.com/writing/3e3ec25bfa464049ae173c31a6d98cf8>

<https://www.jianshu.com/p/5f1630f3bd63>