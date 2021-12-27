关于k8s安装方式的介绍
===================

## kubernetes的安装有几种方式：


##### 使用现成的二进制文件

直接从官方或其他第三方下载，就是kubernetes各个组件的可执行文件。拿来就可以直接运行了。不管是centos，ubuntu还是其他的linux发行版本，只要gcc编译环境没有太大的区别就可以直接运行的。使用较新的系统一般不会有什么跨平台的问题。

##### 使用源码编译安装

编译结果也是各个组件的二进制文件，所以如果能直接下载到需要的二进制文件基本没有什么编译的必要性了。

##### 使用镜像的方式运行

同样一个功能使用二进制文件提供的服务，也可以选择使用镜像的方式。就像nginx，像mysql，我们可以使用安装版，搞一个可执行文件运行起来，也可以使用它们的镜像运行起来，提供同样的服务。kubernetes也是一样的道理，二进制文件提供的服务镜像也一样可以提供。


从上面的三种方式中其实使用镜像是比较优雅的方案，容器的好处自然不用多说。但从初学者的角度来说容器的方案会显得有些复杂，不那么纯粹，会有很多容器的配置文件以及关于类似二进制文件提供的服务如何在容器中提供的问题，容易跑偏。 所以我们这里建议: 生产环境使用二进制方式安装，测试环境可以使用镜像方式。



## 关于k8s的证书
kubernetes 系统各组件需要使用 TLS 证书对通信进行加密，使用 CloudFlare 的 PKI 工具集生成自签名的 CA 证书，用来签名后续创建的其它 TLS 证书。

根据认证对象可以将证书分成三类：服务器证书server cert，客户端证书client cert，对等证书peer cert(既是server cert又是client cert)，在kubernetes 集群中需要的证书种类如下：
* etcd 节点需要标识自己服务的server cert，也需要client cert与etcd集群其他节点交互，当然可以分别指定2个证书，为方便这里使用一个对等证书。

*  master 节点需要标识 apiserver服务的server cert，也需要client cert连接etcd集群，这里也使用一个对等证书。
* kubectl calico kube-proxy 只需要client cert，因此证书请求中 hosts 字段可以为空。
* kubelet 需要标识自己服务的server cert，也需要client cert请求apiserver，也使用一个对等证书。


### certificate_json文件的使用

- 创建 CA 证书
```cfssl gencert -initca ca-csr.json | cfssljson -bare ca```

- 使用CA证书签发kube-apiserer证书
```cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes```

### 生成kubeconfig文件，以kube-controller-manager举例

- 创建 kube-controller-manager证书与私钥

```
cfssl gencert -ca=ca.pem \
-ca-key=ca-key.pem \
-config=ca-config.json \
-profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

- 设置认证参数

```
kubectl config set-credentials system:kube-controller-manager \
---client-certificate=kube-controller-manager.pem \
-client-key=kube-controller-manager-key.pem \
--embed-certs=true \
--kubeconfig=kube-controller-manager.kubeconfig
```

- 设置上下文参数

```
kubectl config set-context default \
--cluster=kubernetes \
--user=system:kube-controller-manager \
--kubeconfig=kube-controller-manager.kubeconfig
```

- 选择默认上下文

```
kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

### 生成kubeconfig 配置文件,用于kubelet认证

- 生成 admin 用户证书
```
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

- 生成 `~/.kube/config` 配置文件
```
kubectl config set-cluster kubernetes --certificate-authority=ca.pem --embed-certs=true --server=127.0.0.1:8443
kubectl config set-credentials admin --client-certificate=admin.pem --embed-certs=true --client-key=admin-key.pem
kubectl config set-context kubernetes --cluster=kubernetes --user=admin
kubectl config use-context kubernetes
```

备注：***使用kubectl config 生成kubeconfig 自动保存到 ~/.kube/config，生成后 cat ~/.kube/config可以验证配置文件包含 kube-apiserver 地址、证书、用户名等信息。***


## 关于k8s组件启动文件的使用，以kube-apiserver举例

##### 复制文件到启动文件目录
cp kube-apiserver.service /lib/systemd/system/kube-apiserver.service

##### 启动服务
systemctl enable kube-apiserver
systemctl start kube-apiserver

##### 查看日志
journalctl -f -u kube-apiserver
