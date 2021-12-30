# 使用nginx对外提供服务

### 说明
现在大部分企业的生产环境已经上云，如何对外提供服务? 可以使用前面介绍的ingress的方式；但是从实际考虑，原生产环境中nginx的配置文件有很多，如果全部改造成ingress的方式，迁移工作量不小。下面主要介绍一下使用nginx对外提供服务。

***要点：就是把nginx配置文件以目录挂载的方式提供给pod***

**1. 新建nginx的清单文件nginx.yaml**

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: monitoring
  labels:
    app: nginx-svc
spec:
  clusterIP: 10.68.80.80
  ports:
  - port: 80
    targetPort: 80
    name: http
  - port: 443
    targetPort: 443
    name: https
  selector:
    app: nginx

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginxconfig
  namespace: monitoring
data:
  nginx.conf: |
    user  root root;
    worker_processes  4;
    worker_rlimit_nofile 104800;
    error_log  /data/logs/nginx/error.log warn;

    pid        /var/run/nginx.pid;

    events {
        use epoll;
        worker_connections 104800;
        accept_mutex off;
    }

    http {
        include       mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr $http_x_forwarded_for - [$time_local] "$request" '
                          '$request_time $upstream_response_time "$upstream_addr" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" $upstream_cache_status';

        access_log  /data/logs/nginx/access.log  main;
        map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
        }

        charset utf-8;
        server_names_hash_bucket_size 128;
        client_header_buffer_size 16k;
        large_client_header_buffers 4 8k;
        client_max_body_size 512m;
        sendfile on;
        sendfile_max_chunk 2048k;
        tcp_nopush on;
        tcp_nodelay on;
        client_header_timeout 300s;
        client_body_timeout 300s;
        send_timeout 300s;
        keepalive_timeout  480;
        keepalive_requests 10000;
        server_tokens off;
        client_body_temp_path /tmp 1 2;
        client_body_buffer_size 16m;
        client_body_in_file_only off;
        chunked_transfer_encoding off;

        gzip on;
        gzip_disable "msie6";
        gzip_proxied any;
        gzip_min_length 1k;
        gzip_buffers 4 16k;
        gzip_http_version 1.1;
        gzip_comp_level 4;
        gzip_types text/plain text/htm text/css application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript application/json;
        gzip_vary on;

        proxy_connect_timeout 600s;
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
        #proxy_next_upstream http_502 http_504 http_404 error timeout invalid_header;
        proxy_redirect off;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header RealURL $scheme://$host$request_uri;
        proxy_buffering on;
        proxy_buffer_size 64k;
        proxy_buffers 16 4m;
        proxy_busy_buffers_size 32m;
        proxy_temp_file_write_size 64m;

        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 16 16k;
        fastcgi_busy_buffers_size 32k;
        fastcgi_temp_file_write_size 64k;

        server
        {
            listen 80 default_server;
            server_name - "";
            access_log /data/logs/nginx/default.log main;

            location / {
                #The location setting lets you configure how nginx responds to requests for resources within the server.
                root   html;
                index  index.html index.htm;
            }

            error_page   500 502 503 504  /50x.html;
            location = /50x.html {
                root   html;
            }

        }


        include /usr/local/nginx/conf.d/*.conf;
    }
---

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: "isnginx"
                operator: In
                values:
                - "nginx"
      tolerations:
      - operator: "Exists"
      #nodeSelector:
      #  kubernetes.io/hostname: cdh-slave.abc.com
      #dnsPolicy: ClusterFirstWithHostNet # 如果使用主机网络，则启用
      #hostNetwork: true                  # 如果使用主机网络，则启用
      hostAliases:
      - ip: 127.0.0.1
        hostnames:
        - "test.abc.com"
      volumes:
      - name: logs
        hostPath:
          path: /data/logs/nginx
          type: DirectoryOrCreate
      - name: nginxconf
        hostPath:
          path: /data/nginx
          type: Directory
      - name: nginxconfig
        configMap:
          name: nginxconfig
      containers:
      - name: nginx
        image: nginx:1.21.1-perl
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: http
          hostPort: 80
        - containerPort: 443
          name: https
          hostPort: 443
        livenessProbe:
          initialDelaySeconds: 20
          failureThreshold: 3
          periodSeconds: 60
          successThreshold: 1
          tcpSocket:
            port: 80
          timeoutSeconds: 1
        resources:
          requests:
            cpu: "500m"
            memory: 4Gi
          limits:
            cpu: "2000m"
            memory: 8Gi
        volumeMounts:
        - name: nginxconfig
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: logs
          mountPath: /data/logs/nginx
        - name: nginxconf
          mountPath: /usr/local/nginx


```

**2. 设置节点标签并应用清单文件**

`kubectl label node cdh-slave.abc.com isnginx=nginx`

`kubectl apply -f nginx.yaml `


**3. 其它的域名的nginx配置文件放到/data/nginx/conf.d目录下面**

**4. 在云上建立负载均衡，把80,443绑定到nginx所在的服务器**


### 清单文件部分说明

```
挂载目录定义
......
volumes:
- name: logs  #定义日志目录
  hostPath:
    path: /data/logs/nginx
    type: DirectoryOrCreate
- name: nginxconf  # 定义ng配置文件目录
  hostPath:
    path: /data/nginx
    type: Directory
......
```

```
网络设置
...
nodeSelector: # 节点选择
  kubernetes.io/hostname: cdh-slave.abc.com
dnsPolicy: ClusterFirstWithHostNet # 如果使用主机网络，则启用。
hostNetwork: true                  # 使用主机网络。
...
```



```
host域名解析
...
hostAliases:  # 设置pod中/etc/hosts文件中域名的解析。
- ip: 127.0.0.1
  hostnames:
  - "test.abc.com"
...
```


```
挂载目录
...
volumeMounts:
- name: nginxconfig
  mountPath: /etc/nginx/nginx.conf
  subPath: nginx.conf  #subPath ：将configMap和secret作为文件挂载到容器中而不覆盖挂载目录下的文件
...
```





### 关于Pod dnsPolicy

在Kubernetes中，可以针对每个Pod设置DNS的策略，通过PodSpec下的dnsPolicy字段可以指定相应的策略，目前支持的策略如下：

- Default: Pod继承所在宿主机的设置，也就是直接将宿主机的/etc/resolv.conf内容挂载到容器中。
- ClusterFirst: 默认的配置，所有请求会优先在集群所在域查询，如果没有才会转发到上游DNS。
- ClusterFirstWithHostNet: 和ClusterFirst一样，不过是Pod运行在hostNetwork:true的情况下强制指定的。
- None: 1.9版本引入的一个新值，这个配置忽略所有配置，以Pod的dnsConfig字段为准。

为什么会想起找一下dnsPolicy的文档呢，也是因为Pod里默认使用了ClusterFirst策略，导致经常有DNS请求出现timeout问题，想用一个简单的办法继承宿主机的配置，现在看来比较简单了，直接设置dnsPolicy:Default就可以了。

针对上面说的dnsConfig字段，也有个详细的说明：

dnsConfig字段包括下面几个属性：

- nameservers: DNS Server的列表，最多3个IP/
- searches: search域名列表，也就是/etc/resolv.conf中的search字段的配置，最多配置6个
- options: u选项列表，也就是/etc/resolv.conf中的option字段的配置


一个测试的yaml

```
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: dns-example
spec:
  containers:
    - name: test
      image: nginx
  dnsPolicy: "None"
  dnsConfig:
    nameservers:
      - 1.2.3.4
    searches:
      - ns1.svc.cluster.local
      - my.dns.search.suffix
    options:
      - name: ndots
        value: "2"
      - name: edns0
```

最后Pod中的/etc/resolv.conf配置就如下：
```
nameserver 1.2.3.4
search ns1.svc.cluster.local my.dns.search.suffix
options ndots:2 edns0
```

**参考：**

https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#pod-s-dns-policy


### 关于subPath

**什么是subPath**

为了支持单一个pod多次使用同一个volume而设计，subpath翻译过来是子路径的意思，如果是数据卷挂载在容器，指的是存储卷目录的子路径，如果是配置项configMap/Secret，则指的是挂载在容器的子路径。


**subPath的使用场景**

1、 1个pod中可以有多个容器，有时候希望将不同容器的路径挂载在存储卷volume的子路径，这个时候需要用到subpath

2、volume支持将configMap/Secret挂载在容器的路径，但是会覆盖掉容器路径下原有的文件，如何支持选定configMap/Secret的每个key-value挂载在容器中，且不会覆盖掉原目录下的文件，这个时候也可以用到subpath
