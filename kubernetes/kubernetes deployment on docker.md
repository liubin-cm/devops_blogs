#注意事项
kubernetes 1.3.7依赖docker 1.10以上版本，需要先升级docker为最新版本。
```
[root@k8s1-master ~]# docker --version
Docker version 1.10.3, build cb079f6-unsupported
```

# kubernetes master setup
## 下载服务端镜像
download apiserver, controller manager, scheduler, 命令行如下:
```
1. docker pull liubin248/kubernetes-apiserver
2. docker pull liubin248/kube-controller-manager
3. docker pull liubin248/kube-scheduler
```

## 创建ca证书
参考：http://kubernetes.io/docs/admin/authentication/
使用openssl的方式进行创建：
```
openssl can also be use to manually generate certificates for your cluster.
1. Generate a ca.key with 2048bit:
  ·openssl genrsa -out ca.key 2048·
2. According to the ca.key generate a ca.crt (use -days to set the certificate effective time):
  ·openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt·
3. Generate a server.key with 2048bit
  ·openssl genrsa -out server.key 2048·
4. According to the server.key generate a server.csr:
  ·openssl req -new -key server.key -subj "/CN=${MASTER_IP}" -out server.csr·
5. According to the ca.key, ca.crt and server.csr generate the server.crt:
  ·openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 10000·
6. View the certificate.
  ·openssl x509  -noout -text -in ./server.crt·
```
上述ca证书生成在/root/openssl目录下。  

## 生成用户名密码
```
echo 123456,admin,qinghua > /root/openssl/basic_auth.csv
```
 
## 启动kubernetes master服务
1 启动apiserver
```
docker run -d --name=apiserver --net=host -v /root/openssl:/security docker.io/liubin248/kubernetes-apiserver kube-apiserver --logtostderr=true --v=0 --etcd-servers=http://127.0.0.1:2379 --insecure-bind-address=0.0.0.0 --kubelet-port=10250 --allow-privileged=false --service-cluster-ip-range=10.254.0.0/16 --admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota --secure-port=443 --tls-cert-file=/security/server.crt --tls-private-key-file=/security/server.key --secure-port=443 --basic-auth-file=/security/basic_auth.csv
```

2 启动controller-manager
```
docker run -d --name=cm -v /root/openssl:/security docker.io/liubin248/kube-controller-manager kube-controller-manager --master=$apiservermaster:8080 --v=0 --root-ca-file=/security/ca.crt --service-account-private-key-file=/security/server.key
```

3 启动scheduler
```
docker run -d --name=scheduler docker.io/liubin248/kube-scheduler kube-scheduler --master=$apiservermaster:8080
```
其中$apiservermaster需要替换为实际ip地址

# kubernetes client setup
## 下载hyperkube镜像并拷贝可执行文件到宿主机
命令：
```
docker pull liubin248/hyperkube
```

## 运行容器并将hyperkube拷贝到客户机
操作步骤如下：
```
1. 运行hyperkube容器
docker run --net=host -d --name=kubeproxy -v /var/lib/docker:/var/lib/docker:rw docker.io/liubin248/hyperkube /hyperkube proxy --master=http://$apiservermaster:8080
2. 解除selinux限制
chcon -Rt svirt_sandbox_file_t /var/lib/docker
3. 进行容器
docker exec -ti kubeproxy /bin/bash
4. 拷贝hyperkube二进制文件
cp /hyperkube /var/lib/docker
```

## 客户机运行hyperkube
```
1. 后台运行kubelet服务
nohup /var/lib/docker/hyperkube kubelet --api_servers=http://$apiservermaster:8080 --pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest --cluster_dns=10.254.210.250 --cluster_domain=cluster.local --hostname_override=$nodeName &
其中nodeName为客户机名字

2. 后台运行proxy服务
/var/lib/docker/hyperkube proxy --master=http://$apiservermaster:8080
```

# kubectl的安装使用
kubectl也可以使用hyperkube里面的指令
```
/var/lib/docker/hyperkube kubectl commands
```
其中commands需要替换为实际参数

# kubenetes trouble shooting
上述安装过程没有将日志输出到标准输出，也没有输出到日志文件，不方便kubernetes故障定位，在实际安装过程中，还需要对日志进行指定输出。

#参考
http://qinghua.github.io/kubernetes-security/
