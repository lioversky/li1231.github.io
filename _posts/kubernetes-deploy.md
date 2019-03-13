# kubernetes安装记录

## 安装docker

参考链接：https://docs.docker.com/install/linux/docker-ce/centos/#set-up-the-repository

1. Install required packages.

```shell
sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
 ``` 
 2. Use the following command to set up the stable repository.

 ```shell
 sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
 ```
 
 3. INSTALL DOCKER CE

```shell
 sudo yum install docker-ce
```

 错误：软件包：3:docker-ce-18.09.0-3.el7.x86_64 (docker-ce-stable)
          需要：container-selinux >= 2.9
 解决：yum install -y http://mirror.centos.org/centos/7/extras/x86_64/Packages/container-selinux-2.74-1.el7.noarch.rpm

过程：
```
导入 GPG key 0x621E9F35:
 用户ID     : "Docker Release (CE rpm) <docker@docker.com>"
 指纹       : 060a 61c5 1b55 8a7f 742b 77aa c52f eb6b 621e 9f35
 来自       : https://download.docker.com/linux/centos/gpg
```

4. start docker

```
sudo systemctl start docker
```

## 安装etcd

略

## 安装kubernetes 

### 下载

1. 源码编译
```
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes
make release
```

2. pre-build
	https://kubernetes.io/docs/setup/release/notes/
	
### 配置

1. config

```
KUBE_LOGTOSTDERR="--logtostderr=true"
KUBE_LOG_LEVEL="--v=0"
KUBE_ALLOW_PRIV="--allow-privileged=false"
KUBE_MASTER="--master=http://10.210.136.30:8080"
```

### 非安全启动


#### 1.kube-controller-manager

```
nohup ./server/bin/kube-apiserver \
	--logtostderr=true \
	--v=0 \
	--etcd-servers=http://10.13.56.79:2379 \
	--insecure-bind-address=0.0.0.0 \
	--insecure-port=8080 \
	--secure-port=6443 \
	--service-node-port-range=1-65535 \
	--allow-privileged=false \
	--service-cluster-ip-range=10.254.0.0/16 \
	--admission-control=NamespaceLifecycle,LimitRanger,DefaultStorageClass,ResourceQuota \
	> logs/apiserver.log 2>&1 &
```

#### 2.controller-manager

```
nohup server/bin/kube-controller-manager \
	--logtostderr=true \
	--v=0 \
	--master=http://10.210.136.30:8080 \
	> logs/ctrl_manager.log 2>&1 &
```

#### 3.kube-scheduler

```
nohup server/bin/kube-scheduler \
	--logtostderr=true \
	--v=0 \
	--master=http://10.210.136.30:8080 \
	> logs/scheduler.log 2>&1 &
```

#### 4.kubelet

```
nohup server/bin/kubelet \
	--logtostderr=true \
	--v=0 \
	--hostname-override=10.210.136.30 \
	--fail-swap-on=false \
    --kubeconfig=bootstrap.kubeconfig \
    --runtime-cgroups=/systemd/system.slice \
    --kubelet-cgroups=/systemd/system.slice \
> logs/kubelet.log 2>&1 &
```

#### 5.kube-proxy

```
nohup node/bin/kube-proxy \
--logtostderr=true \
--v=0 \
--master=http://10.210.136.30:8080 \
> logs/proxy.log 2>&1 &	
```

#### 测试

环境：两台屡有

创建两个nginx，配置如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mynginx1
spec:
  containers:
  - name: mynginx
    image: nginx
    ports:
    - containerPort: 80
      hostPort: 30001
	resources:
      requests:
        memory: "64Mi"
        cpu: "500m"
      limits:
        memory: "128Mi"
        cpu: "1000m"
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mynginx
  labels:
    name: test
spec:
  containers:
  - name: mynginx
    image: nginx
    ports:
    - containerPort: 80
      hostPort: 30001
```

再以NodePort类型创建service，这样创建可以通过nodeIP:nodePort访问，port为上面创建nginx的containerPort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30000
  selector:
    name: test
```

Deployment

```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: alert-manager
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: alert-manager
    spec:
      containers:
      - name: alert
        image: registry.api.weibo.com/dip/alert-manager:v1.0
```


### 安全启动


启动命令

```
nohup server/bin/kube-apiserver   --logtostderr=true   --v=0   --etcd-servers=http://10.13.56.79:2379   --insecure-bind-address=0.0.0.0   --secure-port=6443   --allow-privileged=false   --service-cluster-ip-range=10.254.0.0/16   --admission-control=AlwaysAdmit   --client-ca-file=/etc/kubernetes/ssl/ca.crt   --tls-private-key-file=/etc/kubernetes/ssl/server.key   --tls-cert-file=/etc/kubernetes/ssl/server.crt > logs/apiserver.log 2>&1 &
nohup server/bin/kube-controller-manager --logtostderr=true --v=0 --master=https://10.210.136.30:8080 > logs/ctrl_manager.log 2>&1 &
nohup server/bin/kube-scheduler --logtostderr=true --v=0 --master=http://10.210.136.30:8080 > logs/scheduler.log 2>&1 &
nohup server/bin/kube-proxy --logtostderr=true --v=0 --master=http://10.210.136.30:8080 > logs/proxy.log 2>&1 &

```

#### apiserver

```
bin/kube-apiserver \
	--logtostderr=true \
	--v=0 \
	--etcd-servers=http://10.13.56.79:2379 \
	--insecure-bind-address=0.0.0.0 \
	--secure-port=6443 \
	--allow-privileged=false \
	--service-cluster-ip-range=10.254.0.0/16 \
	--admission-control=AlwaysAdmit \
	--client-ca-file=/etc/kubernetes/ssl/ca.crt \
	--tls-private-key-file=/etc/kubernetes/ssl/server.key \
	--tls-cert-file=/etc/kubernetes/ssl/server.crt

```

#### kube-controller-manager

```
bin/kube-controller-manager \
	--logtostderr=true \
	--v=0 \
	--master=https://10.210.136.30:6443 \
	--kubeconfig=/etc/kubernetes/kubeconfig
	
```

#### kube-scheduler

```
bin/kube-scheduler \
	--logtostderr=true \
	--v=0 \
	--master=http://10.210.136.30:8080
```

### 生成证书
1. CA根证书文件

```
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -days 10000 -subj "/CN=localhost" -out ca.crt
```

2. 生成服务端证书文件和密钥文件
```
openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=localhost" -out server.csr
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 365 -out server.crt
```

3. controller manager客户端证书

```
openssl genrsa -out cs_client.key 2048
openssl req -new -key cs_client.key -subj "/CN=localhost" -out cs_client.csr
openssl x509 -req -in cs_client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out cs_client.crt -days 365
```


参考：https://www.jianshu.com/p/6a6abeefbcbf

#### kube-proxy

```
bin/kube-proxy \
	--logtostderr=true \
	--v=0 \
	--master=http://10.210.136.30:8080
```

#### 生成bootstrap.kubeconfig

生成token，Token可以是任意的包含128 bit的字符串，可以使用安全的随机数发生器生成。

```
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat > conf/token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```

创建 kubelet bootstrapping kubeconfig 文件

```
export KUBE_APISERVER="https://10.210.136.30:8080"

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

在当前目录下会生成bootstrap.kubeconfig

参考：https://jimmysong.io/kubernetes-handbook/practice/create-kubeconfig.html

#### kubelet

```
server/bin/kubelet \
	--logtostderr=true \
	--v=0 \
	--hostname-override=localhost \
	--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest \
	--fail-swap-on=false \
	--kubeconfig=bootstrap.kubeconfig
```




异常

启动kubelet出错：
Failed to get system container stats for "/user.slice/user-1026.slice/session-31286.scope": failed to get cgroup stats for "/user.slice/user-1026.slice/session-31286.scope": failed to get container info for "/user.slice/user-1026.slice/session-31286.scope": unknown container "/user.slice/user-1026.slice/session-31286.scope"

解决：
在kubelet中追加配置
--runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice