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
