#环境
Centos 7
python 2.7

#注意事项
kubernetes 1.2.6依赖docker，请使用docker官方推荐版本。

#安装docker官方版本
```
	   # yum remove docker-common docker docker-selinux docker-forward-journald -y
 
       # vim /etc/yum.repos.d/docker.repo
       [dockerrepo]
       name=Docker Repository
       baseurl=https://yum.dockerproject.org/repo/main/centos/7/
       enabled=1
       gpgcheck=1
       gpgkey=https://yum.dockerproject.org/gpg
	   # yum install docker.x86_64 docker-engine.x86_64 docker-selinux.x86_64 docker-common.x86_64
	   # systemctl enable docker.service && systemctl restart docker.service
```
#安装supervisord
使用ansible安装在centos 7上安装supervisord
##playbook supervisor.yml
将[role](https://github.com/liubin-cm/ansible-role-supervisor) 下载到 /etc/ansible/roles/juwai.supervisor
```
- hosts: k8-master
  roles:
    - juwai.supervisor
```
命令行执行
```
ansible-playbook supervisor.yml
```

#安装kubernetes master cluster
下载[role](https://github.com/liubin-cm/k8s-ha-availability.git)到/etc/ansible/roles/liubin.k8s
编写playbook ansible-k8s.yml
```
- hosts: k8-master
  roles:
    - liubin.k8s
```
命令行执行
```
ansible-playbook ansible-k8s.yml
```

#配置haproxy
使用haproxy做apiserver负载均衡
proxy配置模板如下：
```
global
        stats timeout 30s
        daemon

        # Default SSL material locations

        # Default ciphers to use on SSL-enabled listening sockets.
        # For more information, see ciphers(1SSL).

defaults
        log     global
        mode    http
        option  httplog
        option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000

frontend localnodes
    bind *:80
    mode http
    default_backend nodes

backend nodes
    mode http
    balance roundrobin
    option forwardfor
    http-request set-header X-Forwarded-Port %[dst_port]
    http-request add-header X-Forwarded-Proto https if { ssl_fc }
    option httpchk HEAD / HTTP/1.1\r\nHost:localhost
    server web01 10.47.0.1:8080 check
    server web02 10.47.0.2:8080 check
    server web03 10.47.0.3:8080 check
```
