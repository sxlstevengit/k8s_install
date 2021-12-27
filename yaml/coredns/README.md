## 集群DNS

DNS 是 k8s 集群首要部署的组件，它为集群中的其他 pods 提供域名解析服务；主要可以解析 集群服务名 SVC 和 Pod hostname；目前建议部署 coredns。

### 安装DNS
`kubectl apply -f  coredns.yaml`

### 验证 dns服务

- 新建一个测试nginx服务
`kubectl run nginx --image=nginx --expose --port=80`


- 确认nginx服务
```
[root@cdh-master kube-lb]# kubectl get pod,svc|grep nginx
pod/nginx   1/1     Running   0          22s
service/nginx        ClusterIP   10.68.200.103   <none>        80/TCP    22s
```

- 测试pod alpine

kubectl run test --rm -it --image=alpine /bin/sh
If you don't see a command prompt, try pressing enter.

```
/ # nslookup default.svc.cluster.local
Server:         10.68.0.2
Address:        10.68.0.2:53



/ # nslookup nginx.default.svc.cluster.local
Server:         10.68.0.2
Address:        10.68.0.2:53


Name:   nginx.default.svc.cluster.local
Address: 10.68.147.139

/ # nslookup nginx.default.svc.cluster.local
Server:         10.68.0.2
Address:        10.68.0.2:53

Name:   nginx.default.svc.cluster.local
Address: 10.68.200.103


/ # nslookup www.baidu.com
Server:         10.68.0.2
Address:        10.68.0.2:53

Non-authoritative answer:
www.baidu.com   canonical name = www.a.shifen.com

Non-authoritative answer:
www.baidu.com   canonical name = www.a.shifen.com
Name:   www.a.shifen.com
Address: 180.101.49.11
Name:   www.a.shifen.com
Address: 180.101.49.12
```

### 问题

1. 如果你使用calico网络组件，安装完集群后，直接安装dns组件，可能会出现如下BUG，分析是因为calico分配pod地址时候会从网段的第一个地址（网络地址）开始，详见提交的[ISSUE #1710](https://github.com/projectcalico/calico/issues/1710) , 临时解决办法为手动删除POD，重新创建后获取后面的IP地址

 `kubectl delete pod -n kube-system kube-dns-69bf9d5cc9-c68mw`

2. 使用 kubectl run test -it --rm --image=busybox /bin/sh 进行解析测试可能会失败, busybox内的nslookup程序有bug, 详见 https://github.com/kubernetes/dns/issues/109
