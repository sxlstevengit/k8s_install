## Metrics Server简介

从 v1.8 开始，资源使用情况的度量（如容器的 CPU 和内存使用）可以通过 Metrics API 获取；前提是集群中要部署 Metrics Server，它从Kubelet 公开的Summary API采集指标信息。


它符合k8s的监控架构设计，受heapster项目启发，并且比heapster优势在于：访问不需要apiserver的代理机制，提供认证和授权等；很多集群内组件依赖它（HPA,scheduler,kubectl top），因此它应该在集群中默认运行；部分k8s集群的安装工具已经默认集成了Metrics Server的安装，以下概述下它的安装：

- metric-server是扩展的apiserver，依赖于kube-aggregator，因此需要在apiserver中开启相关参数。
- 需要在集群中运行deployment处理请求。



### 前提

1. 设置apiserver相关参数

```
... # 省略
  --requestheader-client-ca-file={{ ca_dir }}/ca.pem \
  --requestheader-allowed-names=aggregator \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --proxy-client-cert-file={{ ca_dir }}/aggregator-proxy.pem \
  --proxy-client-key-file={{ ca_dir }}/aggregator-proxy-key.pem \
  --enable-aggregator-routing=true \
```

2. 生成aggregator proxy相关证书,证书申请文件aggregator-proxy-csr.json在certificate_json目录



### 安装

`kubectl apply -f metrics-server.yaml`


### 验证

- 查看生成的新api：v1beta1.metrics.k8s.io
```
[root@cdh-master calico]# kubectl get apiservice|grep metrics
v1beta1.metrics.k8s.io                  kube-system/metrics-server   True        32m
```
- 查看kubectl top命令（无需额外安装heapster）
```
[root@cdh-master calico]# kubectl top node
NAME                    CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
cdh-master.rongyi.com   173m         4%     1915Mi          52%
cdh-slave.rongyi.com    149m         3%     3352Mi          43%
cdh-slave2.rongyi.com   180m         9%     3859Mi          57%    
```
