## Ingress简介


 K8s集群对外暴露服务的方式目前只有三种：loadblancer、nodeport、ingress。前两种熟悉起来比较快，而且使用起来也比较方便，在此就不进行介绍了。

下面详细讲解下ingress这个服务，ingress由两部分组成：ingress controller和ingress服务。

- 其中ingress controller目前主要有两种：基于nginx服务的ingress controller和基于traefik的ingress controller。

- 其中traefik的ingress controller，目前支持http和https协议。由于对nginx比较熟悉，而且需要使用TCP负载，所以在此我们选择的是基于nginx服务的ingress controller。

- 但是基于nginx服务的ingress controller根据不同的开发公司，又分为k8s社区的ingres-nginx和nginx公司的nginx-ingress。

在此根据github上的活跃度和关注人数，我们选择的是k8s社区的ingres-nginx。

k8s社区提供的ingress，github地址如下：

https://github.com/kubernetes/ingress-nginx

nginx社区提供的ingress，github地址如下：

https://github.com/nginxinc/kubernetes-ingress


## ingress的工作原理
ingress具体的工作原理如下:

ingress contronler通过与k8s的api进行交互，动态的去感知k8s集群中ingress服务规则的变化，然后读取它，并按照定义的ingress规则，转发到k8s集群中对应的service。而这个ingress规则写明了哪个域名对应k8s集群中的哪个service，然后再根据ingress-controller中的nginx配置模板，生成一段对应的nginx配置。

然后再把该配置动态的写到ingress-controller的pod里，该ingress-controller的pod里面运行着一个nginx服务，控制器会把生成的nginx配置写入到nginx的配置文件中，然后reload一下，使其配置生效。以此来达到域名分配置及动态更新的效果。


- 未配置ingress：
集群外部 -> NodePort -> K8S Service

- 配置ingress:
集群外部 -> Ingress -> K8S Service

注意：ingress 本身也需要部署Ingress controller时使用以下几种方式让外部访问

- 使用NodePort方式
- 使用hostPort方式
- 使用LoadBalancer地址方式


## 部署

**安装**

`kubectl apply -f ingress-nginx.yaml`

**查看**
```
[root@cdh-master ingress-nginx]# kubectl get all -n ingress-nginx
NAME                                            READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-djfcm        0/1     Completed   0          71m
pod/ingress-nginx-admission-patch-8wfgh         0/1     Completed   0          71m
pod/ingress-nginx-controller-77c565c995-rbtzj   1/1     Running     0          71m

NAME                                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.68.10.152   <none>        80:32000/TCP,443:31773/TCP   71m
service/ingress-nginx-controller-admission   ClusterIP   10.68.80.95    <none>        443/TCP                      71m

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           71m

NAME                                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-77c565c995   1         1         1       71m

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           3s         71m
job.batch/ingress-nginx-admission-patch    1/1           4s         71m
```

## 验证

**创建一个Service和deployment**

`kubectl apply -f deploy-demo.yaml `

deploy-demo.yaml内容如下：
```
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: default
spec:
  selector:
    app: myapp
    release: canary
  ports:
  - name: http
    targetPort: 80
    port: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-dm
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
      release: canary
  template:
    metadata:
      labels:
        app: myapp
        release: canary
    spec:
      containers:
      - name: myapp
        image: sxlsteven/busybox-web:1.0.1
        ports:
        - name: http
          containerPort: 80
```

**创建ingress**

`kubectl apply -f ingress.yaml `

ingress.yaml内容如下：
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  annotations:
    kubernetes.io/ingress.class: nginx # 很重要，这样才能被ingress-controller识别。
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  rules:
    - host: nginx.abc.com
      http:
        paths:
         - path: /
           backend:
             serviceName: myapp
             servicePort: 80

```

**查看**
```
[root@cdh-master ingress-nginx]# kubectl get ingress | grep nginx.abc.com
nginx-ingress     <none>   nginx.abc.com      10.68.10.152   80      87m
```

**访问**

在客户端主机也可以通过修改本机 hosts 文件，如上例子，增加记录：
192.168.10.73	nginx.abc.com

*结果：*
```
<h1>Welcome to my webwite </h1>
<h2>I am version 2.0，欢迎使用</h2>
```

## 遇到的问题
1. ingress-nginx文件中的镜像需要翻墙下载，官方原文件中镜像版本号后面的哈希值删除，不然会一直下载镜像。我已经修改过，如下：
`k8s.gcr.io/ingress-nginx/controller:v1.1.0@sha256:f766669fdcf3dc26347ed273a55e754b427eb4411ee075a53f30718b4499076a`

2. ingress-nginx中修改ingress-nginx-controller的Service的类型：
  `由#type: LoadBalancer 改成 type: NodePort`
