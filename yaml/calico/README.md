k8s中安装calico要点
====


说明：
----
calico存储方式有两种： "datastore_type": "kubernetes"  和 etcd.  <br>
我的k8s版本： node:v3.21.2版本，  K8s版本：v1.20.8  <br>
官网都有详细介绍：官方推荐是"datastore_type": "kubernetes"的方式，还要注意calico版本和k8s版本的兼容性，官方文档上有详细介绍  <br>

文件介绍：
* calico-datastore-k8s.yaml.official   官方版calico存储方式为k8s
* calico-datastore-k8s.yaml            官方版修改之后直接可用
* calico-etcd.yaml.official            官方版calico存储方式为etcd
* calico-etcd.yaml                     官方版修改之后直接可用
* calico-etcd-cluster-mine.yaml        我公司测试环境用的



## kubernetes的方式，注意修改几个地方，才能正常运行：


`CALICO_IPV4POOL_CIDR： pod的网段，改成自己的. `  <br>
`"kubeconfig": "__KUBECONFIG_FILEPATH__"： 改成自己主机上实际的路径. 如： "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"  `

由于calico是自动检查主机的IP地址的，它不能正确识别那个是主机的出口网卡造成的，可以采用下面的方式来直接指定。<br>

calico会从部署节点路由中获取到达目的ip或者域名的源ip地址 <br>
```
name: IP_AUTODETECTION_METHOD     <br>
value: "can-reach=192.168.10.73"  <br>

#Using domain names
IP_AUTODETECTION_METHOD=can-reach=www.google.com
IP6_AUTODETECTION_METHOD=can-reach=www.google.com

或者指定网卡名称
- name: IP_AUTODETECTION_METHOD
  value: "interface=eth0"

IP_AUTODETECTION_METHOD=interface=eth.*
IP6_AUTODETECTION_METHOD=interface=eth.*

或者: 指定网段
IP_AUTODETECTION_METHOD=cidr=10.0.1.0/24,10.0.2.0/24
IP6_AUTODETECTION_METHOD=cidr=2001:4860::0/64


不然会报错：
K8S网络异常 calico/node is not ready: BIRD is not ready: BGP not established
bird: device1: State changed to up
bird: direct1: State changed to up
bird: Mesh_172_18_0_1: Received: Connection rejected
bird: Mesh_172_18_0_1: Received: Connection rejected
bird: Mesh_172_18_0_1: Received: Connection rejected
```

可以看出容器初始化时用了三个容器，分别是upgrade-ipam、install-cni和flexvol-driver.其中upgrade-ipam容器的作用是查看是否有/var/lib/cni/networks/k8s-pod-network数据，如果有就把本地的ipam数据迁移到Calico-ipam；install-cni是cni-plugin项目里编译出来的一个二进制文件，用来拷贝二进制文件到各个主机的/opt/cni/bin下面的，并生成calico配置文件拷贝到/etc/cni/net.d下面；flexvol-driver使用的镜像是pod2daemon-flexvol，它的作用是 Adds a Flex Volume Driver that creates a per-pod Unix Domain Socket to allow Dikastes to communicate with Felix over the Policy Sync API.如果容器初试化错误，查看calico-node看不出问题，但可以通过查看这三个容器的log来分析，


## etcd方式
生成密钥，过程省略,参考cfssl命令使用。注意虽然官方文件中，写的是etcd证书，但是我是用CA签发的calico证书，此处用calico证书。
* cat calico.pem | base64 -w 0 > etcd-cert
* cat calico-key.pem | base64 -w 0 > etcd-key
* cat ca.pem | base64 -w 0 > etcd-ca

修改calico清单文件，黄色部分

calico-etcd-secrets是secret的方式；也可以手动直接生成secret

手动生成命令：
```
kubectl create secret generic -n kube-system calico-etcd-secrets \
--from-file=etcd-ca=ca.pem \
--from-file=etcd-key=calico-key.pem \
--from-file=etcd-cert=calico.pem
```


```
---
#Source: calico/templates/calico-etcd-secrets.yaml
#The following contains k8s Secrets for use with a TLS enabled etcd cluster.
#For information on populating Secrets, see http://kubernetes.io/docs/user-guide/secrets/
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  # Populate the following with etcd TLS configuration if desired, but leave blank if
  # not using TLS for etcd.
  # The keys below should be uncommented and the values populated with the base64
  # encoded contents of each file that would be associated with the TLS data.
  # Example command for encoding a file contents: cat <file> | base64 -w 0
  etcd-key: 填写上面的加密字符串
  etcd-cert: 填写上面的加密字符串
  etcd-ca: 填写上面的加密字符串
..........
#Datastore: etcd, using Typha is redundant and not recommended.
#Kubeasz uses cmd-line-way( kubectl create) to create etcd-secrets, see more in 'roles/calico/tasks/main.yml'


#Source: calico/templates/calico-config.yaml
#This ConfigMap is used to configure a self-hosted Calico installation.
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  #Configure this with the location of your etcd cluster.
  etcd_endpoints: "https://192.168.10.73:2379,https://192.168.10.74:2379,https://192.168.10.77:2379"
  #etcd_endpoints: "https://192.168.10.73:2379"
  #If you're using TLS enabled etcd uncomment the following.
  #You must also populate the Secret below with these files.
  etcd_ca: "/calico-secrets/etcd-ca"   #这三个值不需要修改
  etcd_cert: "/calico-secrets/etcd-cert" #这三个值不需要修改
  etcd_key: "/calico-secrets/etcd-key"   #这三个值不需要修改
.............

{
      "name": "k8s-pod-network",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "calico",
          "log_level": "info",
          "log_file_path": "/var/log/calico/cni/cni.log",
          "etcd_endpoints": "https://192.168.10.73:2379,https://192.168.10.74:2379,https://192.168.10.77:2379",   # 这个地方不用修改，采用默认值也可，会自动读取变量的值。
          "etcd_key_file": "/etc/calico/ssl/calico-key.pem",  # 这个地方不用修改，采用默认值也可，会自动读取变量的值
          "etcd_cert_file": "/etc/calico/ssl/calico.pem",   # 这个地方不用修改，采用默认值也可，会自动读取变量的值
          "etcd_ca_cert_file": "/etc/kubernetes/ca/ca.pem",  # 这个地方不用修改，采用默认值也可，会自动读取变量的值。
          "mtu": 1500,
          "ipam": {
              "type": "calico-ipam"
          },
          "policy": {
              "type": "k8s"
          },
          "kubernetes": {
              "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
          }
        },
        {
          "type": "portmap",
          "snat": true,
          "capabilities": {"portMappings": true}
        },
        {
          "type": "bandwidth",
          "capabilities": {"bandwidth": true}
        }
      ]
    }
```


IP自动检测
```
name: IP_AUTODETECTION_METHOD
value: "can-reach=192.168.111.129"
```


参考：
https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises#determine-your-datastore
https://www.cnblogs.com/janeysj/p/13752994.html
