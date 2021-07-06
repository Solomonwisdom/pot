# DLKIT on K8s 1.17
#### K8S on ubuntu
```
# step 1: 安装必要1.17.17的一些系统工具
apt-get update
apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | apt-key add -
# Step 3: 写入软件源信息
add-apt-repository "deb [arch=amd64] http://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装 Docker-CE
apt-get -y update
apt install -y docker-ce=5:19.03.15~3-0~ubuntu-bionic docker-ce-cli=5:19.03.15~3-0~ubuntu-bionic
apt-mark hold docker-ce-cli docker-ce

# Add the package repositories
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | \
  apt-key add -
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | \
tee /etc/apt/sources.list.d/nvidia-docker.list
apt-get update

# Install nvidia-docker2 and reload the Docker daemon configuration
apt-get install -y nvidia-docker2
pkill -SIGHUP dockerd
cat << EOF > /etc/docker/daemon.json
{
    "exec-opts": ["native.cgroupdriver=systemd"],
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
EOF
systemctl daemon-reload
systemctl restart docker

/sbin/modprobe -- ip_vs
/sbin/modprobe -- ip_vs_rr
/sbin/modprobe -- ip_vs_wrr
/sbin/modprobe -- ip_vs_sh
/sbin/modprobe -- nf_conntrack
/sbin/modprobe -- nf_conntrack_ipv4
/sbin/modprobe -- nf_conntrack_ipv6
lsmod | grep -e ip_vs -e nf_conntrack

cat <<EOF >> /etc/modules
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
nf_conntrack_ipv4
nf_conntrack_ipv6
br_netfilter
EOF

apt install  -y ipvsadm ipset

apt-get install -y apt-transport-https curl
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt update
apt install -y kubeadm=1.17.17-00 kubelet=1.17.17-00 kubectl=1.17.17-00
apt-mark hold kubelet kubeadm kubectl

swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
cat <<EOF >> /etc/sysctl.conf 
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
vm.swappiness = 0
EOF

sysctl -p

systemctl enable --now kubelet
```
#### K8S on centos
```shell
# 同步时间
yum install -y ntpdate
ntpdate ntp.nju.edu.cn

# 安装docker
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
yum-config-manager \
    --add-repo \
    https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 
yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
yum install -y docker-ce-19.03.14 docker-ce-cli-19.03.14 --disableexcludes=docker-ce-stable
systemctl enable docker && systemctl start docker
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)

curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.repo | sudo tee /etc/yum.repos.d/nvidia-docker.repo
yum clean expire-cache

yum install nvidia-container-toolkit -y
yum install nvidia-docker2 -y
pkill -SIGHUP dockerd
cat << EOF > /etc/docker/daemon.json
{
    "default-runtime": "nvidia",
    "graph": "/data/docker",
    "exec-opts": ["native.cgroupdriver=systemd"],
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
EOF
cat << EOF > /etc/docker/daemon.json
{
    "graph": "/data/docker",
    "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF
mkdir /etc/systemd/system/docker.service.d/
bash docker-proxy.sh
systemctl daemon-reload
systemctl restart docker

cat <<EOF >> /etc/hosts
210.28.132.167 n167.njuics.cn 
210.28.132.168 n168.njuics.cn
210.28.132.169 n169.njuics.cn
210.28.132.170 n170.njuics.cn
210.28.134.32 n32.njuics.cn
210.28.134.33 n33.njuics.cn
10.0.0.167 n167 
10.0.0.168 n168
10.0.0.169 n169
10.0.0.170 n170
EOF
```
```
cat > /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack_ipv4
br_netfilter
EOF
depmod
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack_ipv4
modprobe br_netfilter

yum install -y ipset ipvsadm
```
```
# 使各个端口可用，先简单起见关闭防火墙
systemctl stop firewalld
systemctl disable firewalld
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
swapoff -a
sed -i 's/.*swap.*/#&/' /etc/fstab

cat <<EOF >  /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
cat <<EOF >> /etc/sysctl.conf 
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
vm.swappiness = 0
EOF
sysctl -p


yum install -y kubeadm-1.17.17-0 kubectl-1.17.17-0 kubelet-1.17.17-0  --disableexcludes=kubernetes
systemctl enable --now kubelet
```

#### 安装kubernetes
``` shell

kubeadm config print init-defaults > kubeadm-config.yaml

# 默认用calico，如果用flannel需要把podSubnet修改成10.244.0.0/16
cat > kubeadm-config.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.17.17
controlPlaneEndpoint: 192.168.8.93:6443
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
networking:
  podSubnet: 192.168.0.0/16
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
EOF

kubeadm config images pull --config kubeadm-config.yaml #先把需要的镜像拉取下来
kubeadm init --config kubeadm-config.yaml --upload-certs
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

kubeadm join [masterIP]:6443 --token 7swq19.7rq2wpo0cv4619dk --discovery-token-ca-cert-hash sha256:2b1a07a4e4a022586d972bc76c6d413ee92a7f1ae5a8dca18776b8b225e5b87a

kubectl taint nodes --all node-role.kubernetes.io/master- # 使master节点变为可调度

# 采用flannel网络插件
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
# 1.17
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
# 1.16
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
# 采用calico网络插件
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml
# 1.16
kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
# 1.17
kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
```
#### nginx-ingress

```shell
# hostNetwork
k create ns ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx  
helm install ingress-nginx ingress-nginx/ingress-nginx \
--namespace ingress-nginx --set rbac.create=true,\
controller.kind=DaemonSet,controller.hostNetwork=true,\
controller.admissionWebhooks.enabled=false \
--set-string controller.nodeSelector."controller\.ingress\.io/enabled"="true",\
controller.config."proxy-body-size"="10240m",\
controller.config."proxy-send-timeout"="60",\
controller.config."proxy-read-timeout"="1800"
# 在n167部署ingress-controller，绑定80端口
k label node n167.njuics.cn controller.ingress.io/enabled=true
```

#### ingress示例
```
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: 10240m
    nginx.ingress.kubernetes.io/proxy-send-timeout: 60s
    nginx.ingress.kubernetes.io/proxy-read-timeout: 1800s
    
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  namespace: ics
  name: ics
spec:
  rules:
  - host: ics.nju.edu.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: ics
          servicePort: 38080
---->
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ics
  namespace: ics
spec:
  rules:
  - host: ics.nju.edu.cn
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ics
            port:
              number: 80
```

#### ceph-csi
```
helm repo add ceph-csi https://ceph.github.io/csi-charts
kubectl create ns ceph-csi-cephfs
cd ceph-csi-cephfs
helm install --namespace "ceph-csi-cephfs" "ceph-csi-cephfs" ceph-csi/ceph-csi-cephfs -f charts/ceph-csi-cephfs/values167.yaml
kubectl apply -f examples/cephfs/secret167.yaml 
kubectl apply -f examples/cephfs/storageclass167.yaml 
kubectl patch storageclass cephfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### nfs
```
helm repo add stable https://charts.helm.sh/stable
helm install nfs-client-provisioner stable/nfs-client-provisioner \
--namespace storage --set nfs.server=210.28.167 \
--set nfs.path=/share/ 

# kubectl patch storageclass nfs-client -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### cephfs（deprecated）

```shell
mkdir -p ceph/cephfs/deploy/rbac && cd ceph/cephfs/deploy/rbac
wget https://github.com/kubernetes-incubator/external-storage/raw/master/ceph/cephfs/deploy/rbac/clusterrole.yaml
wget https://github.com/kubernetes-incubator/external-storage/raw/master/ceph/cephfs/deploy/rbac/clusterrolebinding.yaml
wget https://github.com/kubernetes-incubator/external-storage/raw/master/ceph/cephfs/deploy/rbac/deployment.yaml
wget https://github.com/kubernetes-incubator/external-storage/raw/master/ceph/cephfs/deploy/rbac/role.yaml
wget https://github.com/kubernetes-incubator/external-storage/raw/master/ceph/cephfs/deploy/rbac/rolebinding.yaml
wget https://github.com/kubernetes-incubator/external-storage/raw/master/ceph/cephfs/deploy/rbac/serviceaccount.yaml

ceph auth get-key client.admin > /tmp/secret
kubectl create ns cephfs
kubectl create secret generic ceph-secret-admin --from-file=/tmp/secret --namespace=cephfs
cd ../
NAMESPACE=cephfs
kubectl create ns $NAMESPACE
sed -r -i "s/namespace: [^ ]+/namespace: $NAMESPACE/g" ./rbac/*.yaml
sed -r -i "N;s/(name: PROVISIONER_SECRET_NAMESPACE.*\n[[:space:]]*)value:.*/\1value: $NAMESPACE/" ./rbac/deployment.yaml
kubectl -n $NAMESPACE apply -f ./rbac

cat << EOF > cephfs-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: cephfs
  selfLink: /apis/storage.k8s.io/v1/storageclasses/cephfs
parameters:
  adminId: admin
  adminSecretName: ceph-secret-admin
  adminSecretNamespace: cephfs
  claimRoot: /dlkit-volumes
  monitors: 114.212.189.147:6789,114.212.189.125:6789,114.212.189.126:6789
provisioner: ceph.com/cephfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
EOF
kubectl apply -f cephfs-sc.yaml -n cephfs
```
##### openldap
```shell
k apply -f docker-openldap/example/kubernetes/simple
k apply -f docker-phpLDAPadmin/example/kubernetes
# 访问210.28.132.167:31111，导入ldapbackup.ldif
k create -f dex/dex.yaml
```

##### nvidia-device-plugin
```
kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.7.2/nvidia-device-plugin.yml

helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update
helm install \
    --namespace "kube-system" \
    --version=0.7.1 \
    --generate-name \
    --set compatWithCPUManager=true \
    --set migStrategy=mixed \
    --set resources.requests.cpu=100m \
    --set resources.limits.memory=512Mi \
    nvdp/nvidia-device-plugin
```

##### node-exporter
见 https://github.com/Solomonwisdom/k8s-deploy/blob/master/gpu-node-exporter-daemonset.yaml
```shell
# nvidia-driver<=450
kubectl label node --all hardware-type=NVIDIAGPU
kubectl apply -f gpu-node-exporter-daemonset.yaml
```

#### kubeflow
```
vim /etc/kubernetes/manifests/kube-apiserver.yaml
# 加上
    - --service-account-issuer=kubernetes.default.svc
    - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key

kfctl apply -V -f kfctl_k8s_istio.v1.1.0.yaml
```

##### promethueus
```shell
# 先创建node-exporter
# set the default runtime to nvidia by editing /etc/docker/daemon.json.
# https://github.com/NVIDIA/gpu-monitoring-tools/tree/master/exporters/prometheus-dcgm
# helm3
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus --namespace \
prometheus --set nodeExporter.enabled=false,server.persistentVolume.storageClass=cephfs,alertmanager.persistentVolume.storageClass=cephfs,\
server.ingress.enabled=true,server.ingress.hosts={prometheus.njuics.cn},\
server.securityContext.runAsNonRoot=false,server.securityContext.runAsUser=0,\
server.securityContext.runAsGroup=0,server.securityContext.fsGroup=0
```
##### dlkit
```shell
cd dlkit-operator
cp ./cluster/dlkit/values.yaml.tmpl ./cluster/dlkit/values.yaml
helm install --namespace dlkit dlkit ./cluster/dlkit -f ./cluster/dlkit/values.yaml
```

#####  删除节点
```
kubeadm reset -f
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
ipvsadm -C
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /etc/cni/
rm -rf /var/lib/calico
rm -rf /var/run/nodeagent
rm -rf /usr/libexec/kubernetes/kubelet-plugins/volume/exec/nodeagent~uds
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig vxlan.calico down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
ip link delete vxlan.calico
rm -rf /var/lib/etcd/
rm -rf /etc/kubernetes/
systemctl start docker
systemctl start kubelet

kubectl drain [nodeName] --delete-local-data --force --ignore-daemonsets
kubectl delete node [nodeName]
```