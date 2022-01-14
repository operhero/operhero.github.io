---
layout:     post
title:      "StatefulSet"
subtitle:   ""
date:   2022-01-12
author: "qijun"
tags:
- k8s
---

### 有状态应用
1、**拓扑状态**：应用的多个实例之间不是完全对等的。这些应用实例必须按照某种顺序启动。并且，新创建出来的Pod必须和原来Pod的网络表示一致，这样原先的访问者才能使用同样的方法访问到这个新Pod。

2、**存储状态**：应用的多个实例分别绑定了不同的存储数据，他们之间应该保持固定的映射关系。

### Service
Service是Kubernetes项目中用来将一组Pod暴露给外界访问的一种机制。有两种方式：

1、以VIP(virtual IP, 虚拟IP)方式

2、以DNS方式

其中DNS方式又分两种处理方式：

1、Normal Service。例如访问“my-svc.my-namespace.svc.cluster.local"解析到的正是my-svc的VIP

2、Headless Service。只需要配置spec.clusterIP: None即可。它所代理的所有Pod的IP地址都会被绑定到如下格式的DNS记录：
```aidl
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
```

### 固定拓扑状态
编写一个StatefulSet的YAML：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    app: nginx

---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.9.1
          ports:
            - containerPort: 80
              name: web
```

```aidl
kubectl create -f svc.yaml

[root@master k8s]# kubectl get pods -w -l app=nginx
NAME    READY   STATUS    RESTARTS   AGE
web-0   1/1     Running   0          53s
web-1   1/1     Running   0          34s
```

```aidl
[root@slave ~]# kubectl run -i --tty --image busybox:1.28.3 dns-test --restart=Never --rm /bin/sh
If you don't see a command prompt, try pressing enter.
/ # nslookup web-0.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.32.0.2 web-0.nginx.default.svc.cluster.local
/ # nslookup web-1.nginx
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.36.0.3 web-1.nginx.default.svc.cluster.local

```

*注意:这里busybox指定了版本，没有使用默认的最新版本。最新版本会报server can't find web-0.nginx: NXDOMAIN，[采坑指南——k8s域名解析coredns问题排查过程](https://zhuanlan.zhihu.com/p/89898164)*

通过这种方法，Kubernetes就成功地将Pod的拓扑状态，按照Pod的“名字+编号”的方式固定下来。此外，Kubernetes还为每一个Pod提供了一个固定且唯一的访问入口，即这个Pod对应的DNS记录。

### 固定存储状态
[通过nfs的方式创建pv](https://operhero.github.io/2022/01/12/pv/) ,在此基础上，修改上节svc.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
    - port: 80
      name: web
  clusterIP: None
  selector:
    app: nginx

---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.9.1
          ports:
            - containerPort: 80
              name: web
          volumeMounts:
            - name: www
              mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

查看pvc
```aidl
[root@master k8s]# kubectl get pvc
NAME        STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
www-web-0   Bound    pv000    1Gi        RWO                           28s
www-web-1   Bound    pv001    1Gi        RWO                           24s
```

验证
```aidl
for i in 0 1; do kubectl exec web-$i -- sh -c 'echo hello $(hostname) > /usr/share/nginx/html/index.html'; done

for i in 0 1; do kubectl exec -it web-$i -- curl localhost; done

#我的nginx没有安装curl指令，所以kubectl describe pod web-0获取IP后直接在宿主机上
[root@master k8s]# curl 10.32.0.2
hello web-0
 
#删除Pod后验证html依然存在
[root@master k8s]# kubectl delete pod -l app=nginx
[root@master k8s]# curl 10.32.0.2
hello web-0
```

总结：  
**StatefulSet的控制器直接管理的是Pod**  
**Kubernetes通过Headless Service为这些有编号的Pod，在DNS服务器中生成带有相同编号的DNS记录**  
**StatefulSet还为每一个Pod分配并创建一个相同编号的PVC**  

### Mysql集群实战
[k8s-mysql-cluster](https://github.com/xiaochaoren/k8s-mysql-cluster.git)

```
kubectl label nodes master mysql=master
kubectl label nodes slave mysql=slave
mkdir -p /mnt/mysql
kubectl create namespace data

systemctl status kubelet -l #查看报错日志
journalctl -xe #查看报错日志
```