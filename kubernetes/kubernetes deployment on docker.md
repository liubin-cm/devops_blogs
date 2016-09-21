#ע������
kubernetes 1.3.7����docker 1.10���ϰ汾����Ҫ������dockerΪ���°汾��
��[root@k8s1-master ~]# docker --version
Docker version 1.10.3, build cb079f6-unsupported��

# kubernetes master setup
## ���ط���˾���
download apiserver, controller manager, scheduler
commands:
1. docker pull liubin248/kubernetes-apiserver
2. docker pull liubin248/kube-controller-manager
3. docker pull liubin248/kube-scheduler

## ����ca֤��
�ο���http://kubernetes.io/docs/admin/authentication/
ʹ��openssl�ķ�ʽ���д�����
openssl can also be use to manually generate certificates for your cluster.
1. Generate a ca.key with 2048bit:
  ��openssl genrsa -out ca.key 2048��
2. According to the ca.key generate a ca.crt (use -days to set the certificate effective time):
  ��openssl req -x509 -new -nodes -key ca.key -subj "/CN=${MASTER_IP}" -days 10000 -out ca.crt��
3. Generate a server.key with 2048bit
  ��openssl genrsa -out server.key 2048��
4. According to the server.key generate a server.csr:
  ��openssl req -new -key server.key -subj "/CN=${MASTER_IP}" -out server.csr��
5. According to the ca.key, ca.crt and server.csr generate the server.crt:
  ��openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 10000��
6. View the certificate.
  ��openssl x509  -noout -text -in ./server.crt��
����ca֤��������/root/opensslĿ¼�¡�  

## �����û�������
��echo 123456,admin,qinghua > /root/openssl/basic_auth.csv��  
 
## ����kubernetes master����
1. ����apiserver
`docker run -d --name=apiserver --net=host -v /root/openssl:/security docker.io/liubin248/kubernetes-apiserver kube-apiserver --logtostderr=true --v=0 --etcd-servers=http://127.0.0.1:2379 --insecure-bind-address=0.0.0.0 --kubelet-port=10250 --allow-privileged=false --service-cluster-ip-range=10.254.0.0/16 --admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,ResourceQuota --secure-port=443 --tls-cert-file=/security/server.crt --tls-private-key-file=/security/server.key --secure-port=443 --basic-auth-file=/security/basic_auth.csv`
����

2. ����controller-manager
`docker run -d --name=cm -v /root/openssl:/security docker.io/liubin248/kube-controller-manager kube-controller-manager --master=$apiservermaster:8080 --v=0 --root-ca-file=/security/ca.crt --service-account-private-key-file=/security/server.key`

3. ����scheduler
`docker run -d --name=scheduler docker.io/liubin248/kube-scheduler kube-scheduler --master=$apiservermaster:8080`
����$apiservermaster��Ҫ�滻Ϊʵ��ip��ַ

# kubernetes client setup
## ����hyperkube���񲢿�����ִ���ļ���������
���
��docker pull liubin248/hyperkube��

## ������������hyperkube�������ͻ���
�����������£�
1. ����hyperkube����
��docker run --net=host -d --name=kubeproxy -v /var/lib/docker:/var/lib/docker:rw docker.io/liubin248/hyperkube /hyperkube proxy --master=http://$apiservermaster:8080��
2. ���selinux����
`chcon -Rt svirt_sandbox_file_t /var/lib/docker`
3. ��������
`docker exec -ti kubeproxy /bin/bash`
4. ����hyperkube�������ļ�
`cp /hyperkube /var/lib/docker`

## �ͻ�������hyperkube
1. ��̨����kubelet����
`nohup /var/lib/docker/hyperkube kubelet --api_servers=http://$apiservermaster:8080 --pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest --cluster_dns=10.254.210.250 --cluster_domain=cluster.local --hostname_override=$nodeName &`
����nodeNameΪ�ͻ�������

2. ��̨����proxy����
`/var/lib/docker/hyperkube proxy --master=http://$apiservermaster:8080`

# kubectl�İ�װʹ��
kubectlҲ����ʹ��hyperkube�����ָ��
��/var/lib/docker/hyperkube kubectl��

# kubenetes trouble shooting
������װ����û�н���־�������׼�����Ҳû���������־�ļ���������kubernetes���϶�λ����ʵ�ʰ�װ�����У�����Ҫ����־����ָ�������

#�ο�
http://qinghua.github.io/kubernetes-security/