# prometheus

随着heapster项目停止更新并慢慢被metrics-server取代，集群监控这项任务也将最终转移。prometheus的监控理念、数据结构设计其实相当精简，包括其非常灵活的查询语言；但是对于初学者来说，想要在k8s集群中实践搭建一套相对可用的部署却比较麻烦，由此还产生了不少专门的项目（如：[prometheus-operator](https://github.com/coreos/prometheus-operator)）；还有[kube-prometheus](https://github.com/prometheus-operator/kube-prometheus),它集成了前面的prometheus-operator。本文介绍使用kube-prometheus部署集群的prometheus监控。

参考链接:
https://github.com/prometheus-operator/kube-prometheus
https://prometheus-operator.dev/docs/prologue/quick-start/


## 兼容性

参考上面链接中的介绍，由于我的k8s版本是:v1.20.8,所以选择了release-0.7,其它版本有可能不兼容。

kube-prometheus版本 | k8s 1.19 | k8s 1.20 | k8s 1.21 |k8s 1.22|k8s 1.23
------------------ | ----------|--------- |----------|--------|--------
release-0.7  |  OK |   |  OK |   |
release-0.8  |   |  OK | OK  |   |
release-0.9  |   |   | OK  |OK   |
release-0.10 |   |   |   | OK  |  OK
main         |   |   |   | OK  |  OK


## 安装

kubectl create -f manifests/setup
kubectl create -f manifests/

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
http://192.168.10.73:9090
http://192.168.10.73:9093
http://192.168.10.73:3000


## 卸载

`kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup`
