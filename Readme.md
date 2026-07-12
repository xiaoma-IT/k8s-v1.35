# Ubuntu 26.04 完整部署 K8s v1.35 高可用集群全步骤
 # 所有节点统一执行  
 系统基础更新、安装依赖包  
 更新系统源，安装证书、网络、工具依赖  
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl gpg chrony ipset ipvsadm iputils-ping lsb-release gnupg conntrack nftables wget vim net-tools
关闭系统防火墙
  # 关闭系统防火墙
sudo systemctl disable --now ufw 2>/dev/null || true;

# 校验防火墙状态（inactive）
sudo ufw status;
时区与时间同步配置 
 集群节点时间必须一致，否则证书、调度异常  
# 设置上海时区
timedatectl set-timezone Asia/Shanghai

# 配置 chrony 国内 NTP 源
sudo tee /etc/chrony/chrony.conf <<EOF
pool ntp.aliyun.com iburst
pool ntp.tencent.com iburst
driftfile /var/lib/chrony/chrony.drift
log tracking measurements statistics
logdir /var/log/chrony
maxupdateskew 100.0
hwclockfile /etc/adjtime
makestep 1.0 3
rtcsync
EOF

# 重启开机自启并验证时间
sudo systemctl restart chronyd
sudo systemctl enable chronyd

# 验证同步状态
chronyc sources -v
timedatectl
 永久关闭 Swap  
 K8s 强制要求：内存调度冲突会导致 kubelet 崩溃
#临时关闭
sudo swapoff -a

# 永久注释fstab内swap条目，重启不恢复
sudo sed -i '/swap/s/^/#/' /etc/fstab

# 屏蔽swap挂载单元（Ubuntu新版swap.img专用）
sudo systemctl mask swap.img.swap 2>/dev/null || true;

# 校验swap已关闭，Swap总内存必须为0
free -h;

# 验证无swap输出
cat /etc/fstab
swapon --show
 加载网络内核模块 
 overlay/br_netfilter：容器 overlay 网络、桥接流量转发依赖 
# 写入开机自启模块配置
sudo tee /etc/modules-load.d/k8s-network.conf <<EOF
overlay
br_netfilter
EOF

# 立即加载模块
sudo modprobe overlay
sudo modprobe br_netfilter

# 验证模块已加载
lsmod | grep -E "overlay|br_netfilter"
 内核网络转发参数  
 适配 nftables：开启 IP 转发、桥接流量过滤，CNI 网络互通基础  
sudo tee /etc/sysctl.d/99-k8s-nft-forward.conf <<EOF

# 开启IPv4全局转发
net.ipv4.ip_forward = 1

# 桥接流量经过nftables过滤（nftables模式必填）
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# 关闭ipv6（可选，纯ipv4集群）
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
EOF

# 全局生效所有sysctl配置
sudo sysctl --system

# 逐条验证参数是否生效
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
 主机名与 hosts 解析 
 节点间主机名互通，避免 DNS 解析故障  
# 分别设置各节点主机名
sudo hostnamectl set-hostname k8s-master01

# 所有节点统一写入 hosts
sudo tee -a /etc/hosts <<EOF
192.168.1.101 k8s-master01
192.168.1.102 k8s-master02
192.168.1.103 k8s-master03
192.168.1.104 k8s-worker01
192.168.1.105 k8s-worker02
192.168.1.200 k8s-vip
EOF

# 验证连通性（所有节点互相ping主机名、ping VIP）
ping k8s-master01 -c 3
ping k8s-vip -c 3
 安装并完整配置 containerd v2.2.5
# 安装 containerd
sudo apt install -y containerd

# 生成默认配置文件
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml >/dev/null

修改核心配置
# 1. 修改cgroup驱动为systemd（kubelet强制匹配，否则初始化失败）
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 2. 配置镜像加速目录certs.d
sudo sed -i 's#config_path = ""#config_path = "/etc/containerd/certs.d"#' /etc/containerd/config.toml

# 3. 替换pause沙箱镜像为阿里云国内源（避免拉取registry.k8s.io超时）
sudo sed -i 's|sandbox = "registry.k8s.io/pause:3.10"|sandbox = "registry.aliyuncs.com/google_containers/pause:3.10"|' /etc/containerd/config.toml

配置 docker.io 国内镜像加速
# 创建镜像加速目录
sudo mkdir -p /etc/containerd/certs.d/docker.io

# 写入多加速源配置
sudo tee /etc/containerd/certs.d/docker.io/hosts.toml <<EOF
server = "https://docker.io"
[host."https://docker.m.daocloud.io"]
capabilities = ["pull", "resolve"]
[host."https://registry.cn-hangzhou.aliyuncs.com"]
capabilities = ["pull", "resolve"]
[host."https://dockerpull.cn"]
capabilities = ["pull", "resolve"]
EOF
 配置 k8s 官方镜像仓库
sudo mkdir -p /etc/containerd/certs.d/registry.k8s.io

sudo tee /etc/containerd/certs.d/registry.k8s.io/hosts.toml <<EOF
server = "https://registry.k8s.io"
[host."https://m.daocloud.io/registry.k8s.io"]
capabilities = ["pull", "resolve"]
[host."https://k8s.m.daocloud.io"]
capabilities = ["pull", "resolve"]
[host."https://registry.lank8s.cn"]
capabilities = ["pull", "resolve"]
EOF
重启 containerd 并设置开机自启
# 重启 containerd 并设置开机自启
sudo systemctl daemon-reload
sudo systemctl restart containerd
sudo systemctl enable containerd
# 验证运行正常
systemctl status containerd
安装 crictl 组件
#安装 crictl 组件
apt install cri-tools

# 配置 crictl 工具
cat > /etc/crictl.yaml << 'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 30
debug: false
pull-image-on-create: false
EOF

#  crictl 连通校验
sudo crictl info
 安装 kubeadm/kubelet/kubectl v1.35 
# 添加阿里云 K8s v1.35 apt 源
# 创建密钥目录
sudo mkdir -p /etc/apt/keyrings

# 下载阿里云v1.35签名密钥
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.35/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# 写入apt源配置
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.35/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 更新缓存并锁定安装 v1.35.0 版本
sudo apt update

# 精准安装1.35.0
sudo apt install -y kubelet=1.35.0-1.1 kubeadm=1.35.0-1.1 kubectl=1.35.0-1.1

# 锁定版本，防止apt upgrade自动升级
sudo apt-mark hold kubelet kubeadm kubectl

# 验证版本
kubeadm version
kubectl version --client
kubelet --version
master01 执行
 kube-vip 必须在 kubeadm init 前部署，作为静态 Pod 监听 VIP，提供 API Server 负载均衡  
 拉取 kube-vip 镜像  
sudo crictl pull ghcr.io/kube-vip/kube-vip:v1.2.1
 生成 kube-vip 静态 Pod yaml  
 替换参数：VIP、网卡 ens33  
export VIP=192.168.120.100
export INTERFACE=ens33

sudo mkdir -p /etc/kubernetes/manifests

sudo crictl run --rm --net-host ghcr.io/kube-vip/kube-vip:v1.2.1 kube-vip-gen /kube-vip manifest pod \
--interface ${INTERFACE} \
--vip ${VIP} \
--vip-subnet-length 24 \
--controlplane \
--services \
--arp \
--leaderElection | sudo tee /etc/kubernetes/manifests/kube-vip.yaml


参数说明：
--controlplane：为 k8s 控制平面 API 提供 VIP 转发
--services：集群 Service 负载均衡（可选保留）
--arp：ARP 模式，无需交换机配置，适配大多数环境
--leaderElection：多 master 选主，故障自动漂移 VIP
--vip-subnet-length 24 : kube-vip manifest pod 生成参数对应子网配置
 将 kube-vip.yaml 同步至另外两台 master 节点  
 三台控制平面均运行 kube-vip 静态 Pod，实现 VIP 故障转移。  
# 同步到master02
sudo scp /etc/kubernetes/manifests/kube-vip.yaml root@master02:/etc/kubernetes/manifests/
# 同步到master03
sudo scp /etc/kubernetes/manifests/kube-vip.yaml root@master03:/etc/kubernetes/manifests/
编写 kubeadm 初始化配置文件  
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "192.168.120.5"   # master01地址
  bindPort: 6443
nodeRegistration:
  name: master01   # 确保与主机名一致
  criSocket: unix:///var/run/containerd/containerd.sock
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
kubernetesVersion: v1.35.0   # 改为有效版本 
controlPlaneEndpoint: "192.168.120.100:6443"   # kube-vip地址
imageRepository: registry.aliyuncs.com/google_containers
networking:
  serviceSubnet: "10.96.0.0/12"   # svc 网段地址
  podSubnet: "10.244.0.0/16"     # pod 网段地址
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "nftables"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"

预拉取集群所有组件镜像
#预拉取集群所有组件镜像
sudo kubeadm config images pull --config kubeadm-init.yaml;
# 校验镜像全部拉取完成
sudo crictl images | grep registry.aliyuncs.com/google_containers;
 执行集群初始化
sudo kubeadm init --config kubeadm-init-config.yaml --upload-certs | tee kubeadm-init-output.log0;
 kubectl 权限配置（当前用户操作集群）  
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
节点加入集群
master 节点加入集群
kubeadm join 192.168.1.200:6443 --token xxxxxx.xxxxxxxxxxxx \
--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxx \
--control-plane --certificate-key xxxxxxxxxxxxxxxxxxxxx
worker 节点加入集群
kubeadm join 192.168.1.200:6443 --token xxxxxx.xxxxxxxxxxxx \
--discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxx
初始化认证过期重置（可选）
# init-token过期重新生成
## 先上传证书，拿到 certificate-key
kubeadm init phase upload-certs --upload-certs

## 生成永久有效 worker 基础 join 命令
kubeadm token create --ttl 0 --print-join-command

###示例（master）
kubeadm join 192.168.120.100:6443 \
--token iz5fxg.1g3sod1v1hluabxw \
--discovery-token-ca-cert-hash sha256:305b861fa5e83ac3ca66e1566bae643dbeedd639f630035585d92b187ad2e476 \
--control-plane --certificate-key 你的certificate-key
卸载节点（可选）
kubeadm reset -f
rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd
rm -rf /etc/systemd/system/kubelet.service.d /etc/cni/net.d/*
calico 安装
下载 calico 静态 pod 文件
curl -LO https://raw.githubusercontent.com/projectcalico/calico/v3.31.2/manifests/calico.yaml
创建 calico
kubectl apply -f calico.yaml
 集群完整验证  
 验证 kube-proxy 转发模式为 nftables  
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
# 输出mode: nftables 正确
 验证 Calico nftables 数据平面  ( Tigera Operator  才有)
kubectl get installation default -o yaml | grep linuxDataplane
# 输出linuxDataplane: Nftables 正确
 验证 kube-vip VIP 负载均衡  
# 查看kube-vip静态Pod
kubectl get pods -A | grep kube-vip
# 访问VIP 6443端口连通
curl https://192.168.120.100:6443 --insecure
 全组件健康检查  
# 集群控制平面健康
kubectl get cs
kubectl get --raw /healthz

# 所有命名空间Pod无异常
kubectl get pods -A

# 节点资源状态
kubectl top nodes

# 测试创建测试Pod验证网络连通
kubectl run test-pod --image=nginx:alpine
kubectl get pods
kubectl exec -it test-pod -- ping baidu.com -c 3
kubectl delete pod test-pod
补充故障修复命令（常用）  
 重置集群（节点重装）  
sudo kubeadm reset -f
sudo rm -rf /var/lib/etcd /var/lib/kubelet /etc/kubernetes/manifests/*
 重新加载 kube-proxy 配置  
kubectl rollout restart daemonset kube-proxy -n kube-system
 重启 containerd/chrony  
sudo systemctl restart containerd chronyd
 查看 kubelet 日志  
journalctl -u kubelet -f
