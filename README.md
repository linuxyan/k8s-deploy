# kubernetes单master部署(kubeadm)

### 1 网络架构 
```
master       192.168.0.101          server101
worker       192.168.0.102          server102
worker       192.168.0.103          server103
```
#### 设置hostname
```
hostnamectl set-hostname server101 && bash
hostnamectl set-hostname server102 && bash
hostnamectl set-hostname server103 && bash
```
#### 设置hosts文件
```
cat >> /etc/hosts <<EOF
192.168.0.101          server101
192.168.0.102          server102
192.168.0.103          server103
EOF
```
#### 设置免密钥认证(all server)
```
ssh-keygen
ssh-copy-id server101
ssh-copy-id server102
ssh-copy-id server103
```


### 2 Docker安装（all server）

#### 关闭防火墙以及selinux
```
systemctl stop firewalld  && systemctl disable firewalld 
sed -ri '/^[^#]*SELINUX=/s#=.+$#=disabled#' /etc/selinux/config
setenforce 0
```

#### 关闭swap
```
swapoff -a  # 临时关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab #永久关闭
```


#### 设置yum源
```
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo # centos 7

curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-8.repo   # centos 8
```
#### 安装基础软件包
##### Centos 7
```
yum install -y yum-utils device-mapper-persistent-data lvm2 \
wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel  \
openssl-devel curl curl-devel unzip sudo ntp libaio-devel wget vim  \
ncurses-devel autoconf automake zlib-devel python-devel \
epel-release  openssh-server socat ipvsadm conntrack ntpdate telnet
```
##### Centos 8
```
yum install -y yum-utils device-mapper-persistent-data lvm2 \
wget net-tools nfs-utils lrzsz gcc gcc-c++ make cmake libxml2-devel  \
openssl-devel curl curl-devel unzip sudo libaio-devel wget vim  \
ncurses-devel autoconf automake zlib-devel chrony \
epel-release  openssh-server socat ipvsadm conntrack telnet
```


#### 时间同步
```
# Centos 7
echo "0 */1 * * * ntpdate cn.pool.ntp.org" >> /var/spool/cron/root
```
```
# Centos 8
sed -ri 's/^pool*/#&/' /etc/chrony.conf 
sed -ri '3a server ntp.aliyun.com iburst'  /etc/chrony.conf
sed -ri '4a server cn.ntp.org.cn iburst'  /etc/chrony.conf
systemctl enable chronyd
systemctl restart chronyd.service
```

#### 修改内核参数
```
modprobe br_netfilter

cat >> /etc/sysctl.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1
EOF

sysctl -p /etc/sysctl.conf
```
#### 配置ipvs模块
```
cat > /etc/sysconfig/modules/ipvs.modules << "EOF" 
#!/bin/bash
ipvs_modules="ip_vs ip_vs_lc ip_vs_wlc ip_vs_rr ip_vs_wrr ip_vs_lblc ip_vs_lblcr ip_vs_dh ip_vs_sh ip_vs_nq ip_vs_sed ip_vs_ftp nf_conntrack"
for kernel_module in ${ipvs_modules}; do
 /sbin/modinfo -F filename ${kernel_module} > /dev/null 2>&1
 if [ 0 -eq 0 ]; then
 /sbin/modprobe ${kernel_module}
 fi
done
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep ip_vs
```

#### 安装docker
```
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sed -i 's+download.docker.com+mirrors.aliyun.com/docker-ce+' /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce
service docker start && systemctl enable docker
```
#### 设置docker源
```
cat >/etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
        "https://registry.docker-cn.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://dockerhub.azk8s.cn",        
        "http://hub-mirror.c.163.com"
  ],
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

systemctl daemon-reload
systemctl restart docker
systemctl status docker
```

### 3.  安装kubernetes

#### 设置kubernetes的yum源(all server)
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```
#### 安装kublet kubeadmin kubectl(all server)
```
yum install -y kubelet-1.20.11 kubeadm-1.20.11 kubectl-1.20.11
systemctl enable kubelet
```

#### 初始化k8s集群(master)
```
kubeadm init --kubernetes-version=1.20.11 \
--apiserver-advertise-address=192.168.0.101 \
--image-repository registry.aliyuncs.com/google_containers \
--pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=SystemVerification

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 安装calico组件(master)
```
# wget -o calico/calico.yaml https://docs.projectcalico.org/manifests/calico.yaml --no-check-certificate
kubectl apply -f calico/calico.yaml
```

### 查看所有节点和pod状态
```
kubectl get nodes
kubectl get pods -n kube-system

kubectl label node server102 node-role.kubernetes.io/worker=worker  # 设置节点label
kubectl label node server103 node-role.kubernetes.io/worker=worker  # 设置节点label
```

### 设置证书过期时间
```
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep Not  # 查看证书过期时间

# wget -o update-cert/update-kubeadm-cert.sh https://raw.githubusercontent.com/yuyicai/update-kube-cert/master/update-kubeadm-cert.sh

chmod +x update-cert/update-kubeadm-cert.sh  && ./update-cert/update-kubeadm-cert.sh all
```

### 安装dashboard
```
#https://github.com/kubernetes/dashboard/releases
wget -O dashboard_v2.2.0.yaml https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml
kubectl apply -f dashboard_v2.2.0.yaml

kubectl get pods -n kubernetes-dashboard  -o wide	# 查看是否运行成功
kubectl get svc -n kubernetes-dashboard				# 查看网络情况
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard   # 编辑网络为NodePort模式，提供外部访问
# type: NodePort

kubectl get svc -n kubernetes-dashboard				# 查看网络情况
#https://192.168.0.101:port

kubectl create clusterrolebinding dashboard-cluster-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:kubernetes-dashboard

kubectl get secret -n kubernetes-dashboard   # 获取kubernetes-dashboard下面的所有账户

kubectl describe secret kubernetes-dashboard-token-68vqk -n kubernetes-dashboard   # 获取token

#使用获取到的token登录dashboard
```

### 4. 安装metrics-server(资源监控)
```
# wget -o metrics-server/metrics-server.yaml https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# 打开metrics-server.yaml添加如下配置(kubelet 的10250端口使用的是https协议，连接需要验证tls证书，--kubelet-insecure-tls不验证客户端证书)：
# - --kubelet-insecure-tls
# 更换images镜像地址：registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server:v0.5.1

kubectl apply -f metrics-server/metrics-server.yaml
```

###  5. 安装ingress-nginx和metallb(七层代理)
#### 安装ingress-nginx
```
docker load -i ingress-nginx/controller-v1.0.4.tar.gz
docker load -i ingress-nginx/kube-webhook-certgen-v1.1.1.tar.gz

# wget -o ingress-nginx/ingress-nginx-controller.yaml https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.0.4/deploy/static/provider/cloud/deploy.yaml

kubectl apply -f ingress-nginx/ingress-nginx-controller.yaml
```
#### 安装metallb
```
# wget -o metallb/metallb-namespace.yaml https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml 
# wget -o metallb/metallb.yaml https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml

kubectl apply -f metallb/metallb-namespace.yaml
kubectl apply -f metallb/metallb.yaml
kubectl apply -f metallb/metallb-config.yaml # 设置获取ip的范围
```

#### 6 安装nfs-client-provisioner(存储卷)
```
# wget -o nfs-server/rbac.yaml https://raw.githubusercontent.com/kubernetes-sigs/nfs-subdir-external-provisioner/master/deploy/rbac.yaml
# wget -o nfs-server/deployment.yaml https://raw.githubusercontent.com/kubernetes-sigs/nfs-subdir-external-provisioner/master/deploy/deployment.yaml
# wget -o nfs-server/class.yaml https://raw.githubusercontent.com/kubernetes-sigs/nfs-subdir-external-provisioner/master/deploy/class.yaml

----修改deployment.yaml文件设置对应的nfs服务器的ip和目录---
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.0.102
            - name: NFS_PATH
              value: /data/volumes
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.0.102
            path: /data/volumes
------------

kubectl apply -f nfs-server/rbac.yaml
kubectl apply -f nfs-server/deployment.yaml
kubectl apply -f nfs-server/class.yaml

kubectl get StorageClass
```

