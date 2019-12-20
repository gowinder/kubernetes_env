# kubernetes_env
config kubernetes env for some backend services


## install kubernetes

### prepare install in every hosts of the k8s cluster

install docker

`curl -sSL https://get.docker.com | sudo sh`

### host name config in every k8s cluster

#### edit `/etc/hosts`, add host ip like this

```shell
k8s1    192.168.1.171
k8s2    192.168.1.172
k8s3    192.168.1.172
```
#### edit `/etc/hostname` for each host, change it to node hostname

eg: `k8s1`

**do a restart after change hostname**

#### ubuntu

```shell
sudo apt-get update && apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### centos

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable kubelet && systemctl start kubelet
curl -sSL https://get.docker.com | sh
systemctl start docker

```

#### close swap

`swapoff -a` or `vim /etc/fstab`, comment out swap line for permanent

### init master node

#### prepare network

##### flannel

```shell
sysctl net.bridge.bridge-nf-call-iptables=1`
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml --pod-network-cidr=10.244.0.0/16
```

**Note here, can change `--pod-network-cidr` to other private network, I just use default.**

#### start master

`kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=192.168.1.171`

**Note**:

* `--apiserver-advertise-address=` is your master host machine's ip address
* `--pod-network-cidr=` is the same cidr from flannel apply command


#### get join command

after successed init master, it will print 
`Your Kubernetes master has initialized successfully!`, and join command like this:

```shell
kubeadm join 192.168.1.171:6443 --token j2dd0v.wthgew2lc3ph4wal --discovery-token-ca-cert-hash sha256:b7227a816c98c9a3b556e6cf9e4e8bbce2867625eae9b443040d34230bfac0ac
```

record that command for later use.


#### set `kubectl` env

##### for root user

run cmd: `export KUBECONFIG=/etc/kubernetes/admin.conf`, and you may need add this to your ~/.bash_profile or ~/.zshrc...

##### for normal user

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Note: here `$HOME/.kube/config` is a file, not a dir, it contains cert hand master host information, if you don't do these, when you run `kubectl` cmd you will get `The connection to the server localhost:8080 was refused - did you specify the right host or port?` error message.**

##### let master node can also deploy user namespace pods

this is an option when you want your master node can deploy your own pods.
`kubectl taint nodes --all node-role.kubernetes.io/master-`

----

### worker node join k8s cluster

**Note** worker node master be prepare for k8s, need install docker, kubelet, kubeadm kubectl...

run cmd in each node host, change cidr and api server to your own

`kubeadm join 192.168.1.171:6443 --token 77zwc0.zxe9b425dkv13cce --discovery-token-ca-cert-hash sha256:dde580f7321d15b4414f6d945062f9248f61f9a232317dd1396be67b252b9049`

### enable auto start when boot

```shell
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

### check your cluster node

`kubectl get nodes`

#### if you forget record join command

run:

```shell
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
```

it will print a sha256 hash code, use it in join cmd after sha256:....

### kubernetes dashboard

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml`

after that, dashboard are not open outside the cluster, you can use kube proxy: 
`kubectl proxy`, and than visit 'http://master_ip:8001'

but i recommend use nginx for ingress


### nginx

read nginx.md

## some usefull cmd for diagnose

* show kube-system evnents
`kubectl get events --namespace=kube-system`
* show all pods
`kubectl get pods --all-namespaces -o wide`
* show kubelet error, this is usefull when your cluster has failed to up or join master
`systemctl status -l kubelet`
`journalctl -xeu kubelet`

## trouble fix

### 虚拟机注意

* 复制的虚拟机需要修改 `/etc/hostname` 中本机的名称，不能一样了，mac地址也要改
* 在每台虚拟机上的 `/etc/hosts` 里面要写上所有k8s环境虚拟机的域名和地址

### 出现  x509

```shell
Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")`
```

#### 解决

### 出现 Flannel (NetworkPlugin cni) error: /run/flannel/subnet.env: no such file or directory

#### 解决

建立文件 `/run/flannel/subnet.env`
在里面加入,换成自己的子网

```
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=10.244.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
```


### 在客户机上出现 `The connection to the server localhost:8080 was refused - did you specify the right host or port?`

客户机需要 `/etc/kubernetes/admin.conf`, 在master上拷到客户机上才行，要手动拷贝到每个节点主机的 `~/.kube/config` 这个文件去，`config`是文件不是目录