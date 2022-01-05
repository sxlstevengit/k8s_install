# prometheus

随着heapster项目停止更新并慢慢被metrics-server取代，集群监控这项任务也将最终转移。prometheus的监控理念、数据结构设计其实相当精简，包括其非常灵活的查询语言；但是对于初学者来说，想要在k8s集群中实践搭建一套相对可用的部署却比较麻烦，由此还产生了不少专门的项目（如：[prometheus-operator](https://github.com/coreos/prometheus-operator)）；还有[kube-prometheus](https://github.com/prometheus-operator/kube-prometheus),它集成了前面的prometheus-operator。本文介绍使用kube-prometheus部署集群的prometheus监控。

参考链接:

https://github.com/prometheus-operator/kube-prometheus

https://prometheus-operator.dev/docs/prologue/quick-start/


## 兼容性

参考上面链接中的介绍，由于我的k8s版本是:v1.20.8，所以选择了release-0.7，其它版本有可能不兼容。

kube-prometheus版本 | k8s 1.19 | k8s 1.20 | k8s 1.21 |k8s 1.22|k8s 1.23
------------------ | ----------|--------- |----------|--------|--------
release-0.7  |  OK | OK  |   |   |
release-0.8  |   |  OK | OK  |   |
release-0.9  |   |   | OK  |OK   |
release-0.10 |   |   |   | OK  |  OK
main         |   |   |   | OK  |  OK


## 安装
```
kubectl create -f manifests/setup
kubectl create -f manifests/
```

## 访问

**开启端口转发，这种方式只能临时访问。推荐的方式是通过ingress的方式通过域名访问**

**另外端口转发也可以直接修改清单文件**

prometheus

`kubectl --namespace monitoring port-forward --address 0.0.0.0 svc/prometheus-k8s 9090`

alertmanager

`kubectl --namespace monitoring port-forward --address 0.0.0.0 svc/alertmanager-main 9093`

grafana

`kubectl --namespace monitoring port-forward --address 0.0.0.0 svc/grafana 3000`

浏览器直接访问
```
http://192.168.10.73:9090
http://192.168.10.73:9093
http://192.168.10.73:3000
```


## 卸载

`kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup`


## 配置钉钉告警

- **创建钉钉群，获取群机器人 webhook 地址**

使用钉钉创建群聊以后可以方便设置群机器人，【群设置】-【群机器人】-【添加】-【自定义】-【添加】，然后按提示操作即可。
上述配置好群机器人，获得这个机器人对应的Webhook地址，记录下来，后续配置钉钉告警插件要用，格式如下：

`https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxx`


- **创建钉钉告警插件**

参考:

http://theo.im/blog/2017/10/16/release-prometheus-alertmanager-webhook-for-dingtalk/

https://github.com/timonwong/prometheus-webhook-dingtalk

编辑修改dingtalk-webhook.yaml文件中 access_token=xxxxxx 为上一步你获得的机器人认证 token

**安装插件**

`kubectl apply -f dingtalk-webhook.yaml`

- **修改 alertsmanager 告警配置,测试发送告警。**

修改alertmanager-secret.yaml文件，添加如下内容：

```
"receivers":
    - "name": "Default"
    - "name": "Watchdog"
    - "name": "Critical"
    - "name": "dingtalk"
      "webhook_configs":
       - "send_resolved": false
          "url": "http://webhook-dingtalk.monitoring.svc.cluster.local:8060/dingtalk/webhook1/send"
```

告警示例如下：

![dingtalk](https://raw.githubusercontent.com/sxlstevengit/k8s_install/main/img/钉钉报警.png)


## 设置监控kube-controller-manager和kube-scheduler

安装好prometheus之后，点击上方菜单栏Status --- Targets ，我们发现kube-controller-manager和kube-scheduler未发现。

- **确认所有主节点能访问到metrics**

curl 192.168.10.73:10251/metrics

curl 192.168.10.73:10252/metrics

注意：上面两个服务有可能监听的是127.0.0.1,所以需要改成0.0.0.0,且需要重启相应的服务才能生效。


- **创建svc让prometheus自动发现**

因为K8s的这两上核心组件我们是以二进制形式部署的，为了能让K8s上的prometheus能发现，我们还需要来创建相应的service和endpoints来将其关联起来

monitoring-controller-manager-scheduler.yaml

```
apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-controller-manager
  labels:
    k8s-app: kube-controller-manager
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: http-metrics
    port: 10252
    targetPort: 10252
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-controller-manager
  name: kube-controller-manager
  namespace: kube-system
subsets:
- addresses:
  - ip: 192.168.10.73
  - ip: 192.168.10.74
  - ip: 192.168.10.77
  ports:
  - name: http-metrics
    port: 10252
    protocol: TCP

---

apiVersion: v1
kind: Service
metadata:
  namespace: kube-system
  name: kube-scheduler
  labels:
    k8s-app: kube-scheduler
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: http-metrics
    port: 10251
    targetPort: 10251
    protocol: TCP

---
apiVersion: v1
kind: Endpoints
metadata:
  labels:
    k8s-app: kube-scheduler
  name: kube-scheduler
  namespace: kube-system
subsets:
- addresses:
  - ip: 192.168.10.73
  - ip: 192.168.10.74
  - ip: 192.168.10.77
  ports:
  - name: http-metrics
    port: 10251
    protocol: TCP

```




- **应用清单文件且修改servicemonitors**

`kubectl apply -f monitoring-controller-manager-scheduler.yaml`

还要修改servicemonitors

```
# kubectl -n monitoring edit servicemonitors.monitoring.coreos.com kube-scheduler
# 将下面两个地方的https换成http
    port: https-metrics
    scheme: https

# kubectl -n monitoring edit servicemonitors.monitoring.coreos.com kube-controller-manager
# 将下面两个地方的https换成http
    port: https-metrics
    scheme: https

```


## 使用prometheus来监控ingress-nginx

前面部署过ingress-nginx，这个是整个K8s上所有服务的流量入口组件很关键，因此把它的metrics指标收集到prometheus来做好相关监控至关重要，因为前面ingress-nginx服务是以deployment形式部署的，可以映射自己的端口10254到宿主机上，那么我可以直接用pod运行NODE上的IP来看下metrics。

`curl 192.168.10.73:10254/metrics`

**映射端口10254到宿主机上，需要修改ingress-nginx.yaml**
```
apiVersion: v1
kind: Service
......
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: http
      appProtocol: http
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
      appProtocol: https
    - name: metrics
      port: 10254
      protocol: TCP
....


apiVersion: apps/v1
kind: Deployment
......
ports:
  - name: http
    containerPort: 80
    hostPort: 80
    protocol: TCP
  - name: https
    containerPort: 443
    hostPort: 443
    protocol: TCP
  - name: webhook
    containerPort: 8443
    hostPort: 8443
    protocol: TCP
  - name: metrics
    containerPort: 10254
    hostPort: 10254
    protocol: TCP

....
```

**创建 servicemonitor配置让prometheus能发现ingress-nginx的metrics**


servicemonitor-ingress-nginx.yaml内容如下：

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.10
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.1.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-scraping
  namespace: ingress-nginx
spec:
  endpoints:
  - interval: 30s
    path: /metrics
    port: metrics
  jobLabel: app.kubernetes.io/name
  namespaceSelector:
    matchNames:
    - ingress-nginx
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx

```

创建
`kubectl apply -f  servicemonitor-ingress-nginx.yaml`

```
# kubectl -n ingress-nginx get servicemonitors.monitoring.coreos.com
NAME                     AGE
ingress-nginx-scraping   65m
```

**指标一直没收集上来，看看proemtheus服务的日志，发现报错如下**

```
level=error ts=2022-01-05T06:07:36.803Z caller=klog.go:96 component=k8s_client_runtime func=ErrorDepth msg="/app/discovery/kubernetes/kubernetes.go:427: Failed to watch *v1.Service: failed to list *v1.Service: services is forbidden: User \"system:serviceaccount:monitoring:prometheus-k8s\" cannot list resource \"services\" in API group \"\" in the namespace \"ingress-nginx\""

```

**需要修改prometheus的clusterrole**

```
#   kubectl edit clusterrole prometheus-k8s
#------ 原始的rules -------
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get
  -
#---------------------------

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-k8s
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  - nodes/proxy
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - /metrics
  verbs:
  - get

```

再到prometheus UI上看下，发现已经监控到了

ingress-nginx/ingress-nginx-scraping/0 (1/1 up)


## 使用Prometheus来监控二进制部署的ETCD集群

我们是用二进制的方式部署的ETCD集群，并且利用自签证书来配置访问ETCD，正如前面所说，现在关键的服务基本都会留有指标metrics接口支持prometheus的监控。那我们先来查看一下etcd的指标。

`curl --cacert /etc/kubernetes/ca/ca.pem  --cert /etc/kubernetes/ca/etcd/etcd.pem  --key /etc/kubernetes/ca/etcd/etcd-key.pem https://192.168.10.73:2379/metrics`


**配置prometheus发现并监控etcd**

- **把ETCD的证书创建为secret**
`kubectl -n monitoring create secret generic etcd-certs --from-file=/etc/kubernetes/ca/etcd/etcd.pem   --from-file=/etc/kubernetes/ca/etcd/etcd-key.pem    --from-file=/etc/kubernetes/ca/ca.pem`


- **在prometheus里面引用这个secrets**

```
kubectl -n monitoring edit prometheus k8s

spec:
...
  secrets:
  - etcd-certs


  保存退出后，prometheus会自动重启服务pod以加载这个secret配置，过一会，我们进pod来查看下是不是已经加载到ETCD的证书了

  kubectl -n monitoring exec -it prometheus-k8s-0 -c prometheus  -- sh
  /prometheus $ ls /etc/prometheus/secrets/etcd-certs/
  ca.pem        etcd-key.pem  etcd.pem
```


- **创建service、endpoints以及ServiceMonitor的yaml配置**

prometheus-etcd.yaml内容如下：

```
apiVersion: v1
kind: Service
metadata:
  name: etcd-k8s
  namespace: monitoring
  labels:
    k8s-app: etcd
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: api
    port: 2379
    protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd-k8s
  namespace: monitoring
  labels:
    k8s-app: etcd
subsets:
- addresses:
  - ip: 192.168.10.73
  - ip: 192.168.10.74
  - ip: 192.168.10.77
  ports:
  - name: api
    port: 2379
    protocol: TCP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd-k8s
  namespace: monitoring
  labels:
    k8s-app: etcd-k8s
spec:
  jobLabel: k8s-app
  endpoints:
  - port: api
    interval: 30s
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/ca.pem
      certFile: /etc/prometheus/secrets/etcd-certs/etcd.pem
      keyFile: /etc/prometheus/secrets/etcd-certs/etcd-key.pem
      #use insecureSkipVerify only if you cannot use a Subject Alternative Name
      insecureSkipVerify: true
  selector:
    matchLabels:
      k8s-app: etcd
  namespaceSelector:
    matchNames:
    - monitoring
```

创建清单文件

```
# kubectl apply -f prometheus-etcd.yaml
service/etcd-k8s created
endpoints/etcd-k8s created
servicemonitor.monitoring.coreos.com/etcd-k8s created
```

等一会可以在prometheus UI上面看到ETCD集群被监控了

`monitoring/etcd-k8s/0 (3/3 up)`

- **用grafana来展示被监控的ETCD指标**

```
1. 在grafana官网模板中心搜索etcd，下载这个json格式的模板文件
https://grafana.com/dashboards/3070

2.然后打开自己先部署的grafana首页，
点击左边菜单栏四个小正方形方块HOME --- Manage
再点击右边 Import dashboard ---
点击Upload .json File 按钮，上传上面下载好的json文件 etcd_rev3.json，
然后在prometheus选择数据来源
点击Import，即可显示etcd集群的图形监控信息

```
