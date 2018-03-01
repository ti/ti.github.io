## K8S 和 Docker 命令总结

```go
package main

import (
    "log"
    "time"
)

func main() {
    for i :=0; i< 9999999; i++ {
    	log.Println("runing ...", i)
    	time.Sleep(2 * time.Second)
    }
}

```

# K8S 和 Docker 命令总结

准备 Go文件

```
package main

import (
    "log"
    "time"
)

func main() {
    for i :=0; i< 9999999; i++ {
    	log.Println("runing ...", i)
    	time.Sleep(2 * tim.Second)
    }
}
```

编译

```bash
CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o build/release/main ./main.go
```



Dockerfile

```
FROM scratch
COPY build/release/ /app/
CMD ["/app/main"]
```

```

   docker build -t  linx/greeter-srv:static .
   
   
   docker build -t linx/test .
 
  docker images
  
  docker create linx/test
  
  docker start 23c52094d05d
  
  docker ps -a
  
  进入容器
  docker exec -it  344a22aae2fb /bin/bash
  
  
  
  
  #保存
  docker save -o kube-apiserver-amd64.tar gcr.io/google_containers/kube-apiserver-amd64
  
  docker load -i kube-apiserver-amd64.tar
  
  
```



```
Name:                   nginx
Namespace:              default
CreationTimestamp:      Sat, 11 Nov 2017 18:03:08 +0800
Labels:                 run=nginx
Annotations:            deployment.kubernetes.io/revision=1
Selector:               run=nginx
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  1 max unavailable, 1 max surge
Pod Template:
  Labels:  run=nginx
  Containers:
   nginx:
    Image:        nginx:1.7.9
    Port:         <none>
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-665ff4c6f7 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  26m   deployment-controller  Scaled up replica set nginx-665ff4c6f7 to 1
```


Docker 离线转发



```
 sudo yum install -y yum-utils   device-mapper-persistent-data   lvm2
  sudo yum-config-manager --add-repo  https://download.docker.com/linux/centos/docker-ce.repo
  sudo yum-config-manager --enable docker-ce-edge
  sudo yum install docker-ce
  systemctl start docker
  docker run -d -p 5000:5000 --name registry registry:2
  ​
  docker pull gcr.io/google_containers/kubernetes-dashboard-init-amd64:v1.0.1
  docker pull gcr.io/google_containers/kubernetes-dashboard-amd64:v1.7.1
  ​
  ​
  docker tag gcr.io/google_containers/kubernetes-dashboard-amd64:v1.7.1 localhost:5000/google_containers/kubernetes-dashboard-amd64:v1.7.1
  
  docker push localhost:5000/google_containers/kubernetes-dashboard-amd64:v1.7.1
  ​
  docker tag gcr.io/google_containers/kubernetes-dashboard-init-amd64:v1.0.1 localhost:5000/google_containers/kubernetes-dashboard-init-amd64:v1.0.1
  docker push localhost:5000/google_containers/kubernetes-dashboard-init-amd64:v1.0.1
  
  ​
  ## 在自己电脑上:
  ​
  docker pull us.lnx.cm/google_containers/kubernetes-dashboard-init-amd64:v1.0.1
  ​
  docker pull us.lnx.cm/google_containers/kubernetes-dashboard-amd64:v1.7.1
保存

  docker images
  cd ~/
  docker save -o kubernetes-dashboard-init-amd64.tar us.lnx.cm/google_containers/kubernetes-dashboard-init-amd64
  #scp kubernetes-dashboard-init-amd64.tar  node02:~
  ssh root@node2
  docker load -i kubernetes-dashboard-init-amd64.tar
  docker images
  ​
```

视频中的命令：

```
minikube start

minikube  status

#打开dashbaord
minikube  dashboard

# 集群状态
kubectl cluster-info
# API 版本， 管理的所有服务通过API server暴漏出去
kubectl  api-versions

kubectl get nodes

# 注意：执行下面命令之前确保docker 已经pull 了 nginx:1.7.9或已翻墙。 否则 墙很高



#启动一个 名称为 ngnix 的 depolyment （部署）



kubectl run nginx --image=nginx:stable

kubectl run nginx --image=nginx:1.9.1

# kubectl get 资源类型 + 名称

kubectl expose deployment nginx --type=NodePort

kubectl get deploy nginx

# 深层信息

kubectl describe deploy nginx

注意：
# Labels:                 run=nginx  （依靠这个做主要关联）
# Annotations:            deployment.kub
# NewReplicaSet:   nginx-665ff4c6f7 (1/1 replicas created)  保证有一个

kubectl get ReplicaSet

简写 kubectl get rs

# replicaset 包含了 pod
# 可以在pod中进去查看详情
 kubectl get pod  
 
 可以查看运行在那个节点上
 kubectl get pod nginx-665ff4c6f7-plkbc -o wide
 
 
 
 // 如果出现CrashLoopBackOff， 也可以查看日志
 

 就可以查看日志
 
kubectl logs nginx-665ff4c6f7-plkbc

进入  pod （和docker类似）

kubectl exec -it  nginx-cfc4fd5c6-6sz9v /bin/bash
```

文件拷贝：
 docker ps | grep nginx
 
  从docker 拷贝到 宿主机， docker cp c99a51ed4bab:/root/file.txt ./
  
  
  从宿主机拷贝到 docker docker cp file.txt c99a51ed4bab:/root/
  



## 需要访问“部署”， 则需要挂到 service 上

```
app:nginx
```



```yaml
# 类型，相当于 apiservice
kind: Service
apiVersion: v1
metadata:
	name: nginx
	labels:
		app:nginx
sepc:
	ports:
	- name: http
	  port: 8888
	  nodePort:30001
	  targetPort:80
	# 选择 label 是 run=nginx 的 pod
	selector:
	  run: nginx
	# 支持节点外部访问和集群访问
	type: NodePort
	
```

```

# ------------------- Service ------------------- #
kind: Service
apiVersion: v1
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  ports:
    - name: http
      port: 8888
      nodePort: 3005
      targetPort: 80
  selector:
    run: nginx
  type: NodePort
```

```
kubectl create -f nginx.svc.yaml
```

```
查看
kubectl get svc

kubectl get service nginx
```

上面的方法有点复杂， 有其他简单的方法：

minikube service hello-minikube --url

# 直接暴漏nginx部署端口的方法创建service

将nginx 80 端口暴漏出来

```
kubectl expose deploy nginx --type=NodePort --name=nginx-ext --port=80
```

看 deploy 和 和 service 关联起来的

```
kubectl get endpoit  #/ep 
```
发现：两个ip相同

水平扩展直到三个副本

```
kubectl scale deploy nginx --replicas=3
```

查看状态

```
kubectl get deploy nginx

kubectl get rs

kubectl get po

kubectl get ep


发现都是三个了
```

缩小

```
kubectl scale deploy nginx --replicas=2
```



## 升级

```
kubectl set image deply nginx nginx=nginx:1.9.1
```

查看升级状态

```
kubectl rollout status deploy nginx


kubectl rollout history deploy nginx

```

```

kubectl  describe deploy nginx
```



升级失败

kubectl set image deply nginx nginx=nginx:1.95



kubectl rollout status deploy nginx



kubectl rollout history deploy nginx



kubectl rollout history deploy nginx —revision=3

为社么呢

kubectl get rs

kubectl describe rs ngnx-1083445ef



kubecrl get pod

发现pod 出不来

````````
kubectl  describe pod nginx—xx
````````

升级失败回滚

```
kubectl rollout undo deploy nginx
```

```
kubectl get rs
```

```
kubectl get ep
```



kubectl delete po nginx-xxx 

发现他又出现了新的pod

kubectl get po

这个是 k8s的 自愈，弹性伸缩，升级，回滚





···删除

````
kubectl delete svc nginx-ext 
````





## netstat 查看端口占用



```
netstat -tunlp  |grep kube
```





# Iptables对内网开放所有端口，对外只开放80端口

2013-02-24

阅读(196) 评论([0](http://blog.globstudio.com/1248.html#respond))

背景：
准备把Raspberry Pi开放外网访问，先配置好防火墙

目标：
对内网ip开放所有端口
对外网ip只开放80端口
因为是开发环境，所以允许访问外网所有ip和端口

 开放所有端口

```
iptables -P INPUT ACCEPT   
iptables -P OUTPUT ACCEPT 
```





`iptables -F # 清空规则`



`iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT # 允许内网ip访问所有端口`

`iptables -A INPUT -s 127.0.0.1 -d 127.0.0.1 -j ACCEPT # 允许本机本机访问所有端口`

`iptables -A INPUT -p tcp -s 0/0 --dport 80 -j ACCEPT # 允许所有ip访问80端口`

`iptables -A INPUT -m conntrack --ctstate ESTABLISHED -j ACCEPT # 允许本地发起的连接返回数据到任意端口。本地作为客户端可以发起到任何ip和端口的请求，这条规则确保能够收到返回`

`iptables -A OUTPUT  -p tcp --sport 80 -m conntrack --ctstate ESTABLISHED -j ACCEPT # 允许80端口返回数据`

`iptables -A OUTPUT -j ACCEPT #允许返回数据给所有ip所有端口（上面那条就没用了）`

`iptables -P INPUT DROP # 除上面允许的规则，抛弃所有INPUT请求`

`iptables -P FORWARD DROP # 除上面允许的规则，抛弃所有FORWARD请求`

`iptables -P OUTPUT DROP # 除上面允许的规则，抛弃所有OUTPUT请求`

`iptables-save > /etc/iptables/iptables.rules # 保存（ArchLinux系统下的路径可能和其他系统下的路径不同）`

`iptables -L # 列出规则，供查看`



### k8s 安装预备

```
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
```

**關閉 SELinux:**

開啟檔案 /etc/selinux/config:

\# vi /etc/selinux/config

找到以下一行:

SELINUX=enforce

改成:

SELINUX=disabled

另外將 “SELINUXTYPE=targeted” 加上註釋, 改成這樣:

\# SELINUXTYPE=targeted

、临时关闭（不用重启机器）：
setenforce 0     

要重新開機 reboot / restart 後才會套用





安装docker

```
 yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
 
```



````
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
````



```
yum-config-manager --disable docker-ce-edge

yum-config-manager --enable docker-ce-stable
```



```
#找到最新支持的稳定版
yum list docker-ce --showduplicates | sort -r

#安装制定版本
yum install docker-ce-17.06.2.ce
```



# 启动docker

```
systemctl start docker
systemctl enable docker
```



# 剩下的使用kubedm进行安装。



转发

```
sudo yum install -y yum-utils   device-mapper-persistent-data   lvm2
sudo yum-config-manager --add-repo  https://download.docker.com/linux/centos/docker-ce.repo
sudo yum-config-manager --enable docker-ce-edge
sudo yum install docker-ce
systemctl start docker
docker run -d -p 5000:5000 --name registry registry:2

docker pull gcr.io/google_containers/kubernetes-dashboard-init-amd64:v1.0.1
docker pull gcr.io/google_containers/kubernetes-dashboard-amd64:v1.7.1


docker tag gcr.io/google_containers/kubernetes-dashboard-amd64:v1.7.1 localhost:5000/google_containers/kubernetes-dashboard-amd64:v1.7.1
docker push localhost:5000/google_containers/kubernetes-dashboard-amd64:v1.7.1

docker tag gcr.io/google_containers/kubernetes-dashboard-init-amd64:v1.0.1 localhost:5000/google_containers/kubernetes-dashboard-init-amd64:v1.0.1
docker push localhost:5000/google_containers/kubernetes-dashboard-init-amd64:v1.0.1



## 在自己电脑上:

docker pull us.lnx.cm/google_containers/kubernetes-dashboard-init-amd64:v1.0.1

docker pull us.lnx.cm/google_containers/kubernetes-dashboard-amd64:v1.7.1
```

保存

```
docker images
cd ~/
docker save -o kubernetes-dashboard-init-amd64.tar us.lnx.cm/google_containers/kubernetes-dashboard-init-amd64
#scp kubernetes-dashboard-init-amd64.tar  node02:~
ssh root@node2
docker load -i kubernetes-dashboard-init-amd64.tar
docker images

```
