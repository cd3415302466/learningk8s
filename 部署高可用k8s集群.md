# 部署高可用k8s集群



pv、pvc

```bash
nfs-csi
[root@k8s-master01 ~]#kubectl create ns nfs
kubectl create -f https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/example/nfs-provisioner/nfs-server.yaml -n nfs
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.nfs.svc.cluster.local
  share: /
  # csi.storage.k8s.io/provisioner-secret is only needed for providing mountOptions in DeleteVolume
  # csi.storage.k8s.io/provisioner-secret-name: "mount-options"
  # csi.storage.k8s.io/provisioner-secret-namespace: "default"
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4.1

kubectl apply -f deploy/v3.1.0/         #动态制备pvc需要安装3.1.0版本  4.2.0 本人没有耐心等待，所以未必正确

使用3.1.0支持动态制备pvc
```



Ingress-nginx

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.7.0/deploy/static/provider/cloud/deploy.yaml

#需要代理
```

dashborad的前提metrics server

metrics server

```
#部署
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

metrics-server监听每个node的kubelet的10250 10255端口 10250需要认证 cadvisor暴露核心资源占用率 做tls验证。主机名称解析 coredns做名称解析，对应节点

metrics-server报错

```bash
0328 22:06:01.248567  302307 memcache.go:255] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0328 22:06:01.265268  302307 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0328 22:06:01.268921  302307 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0328 22:06:01.272115  302307 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
```

解决方法如下或者搭建dns服务器

metrics server配置文件

```bash
vim metricserver.yaml
    spec:
      hostNetwork: true
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP    #使用内部IP
        - --kubelet-use-node-status-port
        - --kubelet-insecure-tls       #跳过对方证书合法性
        - --metric-resolution=15s
```

有合法的DNS服务器可忽略此步骤

更改以后重新应用

```yaml
1、检查 API Server 是否开启了 Aggregator Routing：查看 API Server 是否具有 --enable-aggregator-routing=true 选项。
[root@master1 ~]# ps -ef | grep apiserver

2、修改每个 API Server 的 kube-apiserver.yaml 配置开启 Aggregator Routing：修改 manifests 配置后 API Server 会自动重启生效。
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.200.3
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --enable-aggregator-routing=true            # 添加本行
 
3、修改metrics-server.yaml文件
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP   # 删掉 ExternalIP,Hostname这两个，这里已经改好了，你那边要自己核对一下
        - --kubelet-use-node-status-port
        - --kubelet-insecure-tls                    #   加上该启动参数
        image: k8s.gcr.io/metrics-server/metrics-server:v0.4.1                 # 镜像地址根据情况修改
        
kubectl apply -f metrics-server.yaml

[root@master1 ~]# kubectl get pod -n kube-system | grep metrics-server
```



dashboard

```shell
#部署
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

#修改
kubectl edit svc ingress-nginx-controller -n ingress-nginx

  externalIPs:
  - 10.0.0.100           #集群可用ip
  
ingress-nginx配合
vim ingress-kubernetes-dashboard.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard
  annotations:
    ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  namespace: kubernetes-dashboard
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /dashboard(/|$)(.*)
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443
        pathType: Prefix

#ingress暴露端口

访问路径https://10.0.0.100/dashboard/#/login

admin.conf文件认证有问题 不可用

#RBAC鉴权机制
创建serviceaccount账号
role 与 rolebinding
[root@k8s-master01 ~]#kubectl create sa dashuser
serviceaccount/dashuser created
[root@k8s-master01 ~]#kubectl create clusterrolebinding dashuser-admin --clusterrole=cluster-admin --serviceaccount=default:dashuser
clusterrolebinding.rbac.authorization.k8s.io/dashuser-admin created

RBAC做分级授权

https://10.0.0.100/dashboard/

token登录

kubectl run client-$RANDOM --image ikubernetes/admin-box:v1.2 --restart=Never -o yaml --dry-run=client > test.yaml
[root@k8s-master01 ~]#vim test.yaml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: client-30908
  name: client-30908
spec:
  serviceAccountName: dashuser
  containers:
  - image: ikubernetes/admin-box:v1.2
    name: client-30908
    command: ['/bin/sh','-c','sleep 999999']
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}

[root@k8s-master01 ~]#kubectl apply -f test.yaml 
pod/client-30908 created
[root@k8s-master01 ~]#kubectl get pod
NAME           READY   STATUS    RESTARTS   AGE
client-30908   1/1     Running   0          6s
[root@k8s-master01 ~]#kubectl exec -it client-30908 -- /bin/bash
root@client-30908 kubernetes.io# cd /var/run/secrets/kubernetes.io/serviceaccount/
root@client-30908 serviceaccount# ls
ca.crt     namespace  token
root@client-30908 serviceaccount# cat token
eyJhbGciOiJSUzI1NiIsImtpZCI6ImVzN09HS0l1QjRfclJmX2o1TDYtMTY5NUR2TGJaMU5TWTFzOHp3VEtxLVEifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzExNTUxNzEwLCJpYXQiOjE2ODAwMTU3MTAsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJkZWZhdWx0IiwicG9kIjp7Im5hbWUiOiJjbGllbnQtMzA5MDgiLCJ1aWQiOiI2MzI1MjY5Yy0yZTg2LTQ1ODctODNmMi04OWZlYzNmMjQ3ZjgifSwic2VydmljZWFjY291bnQiOnsibmFtZSI6ImRhc2h1c2VyIiwidWlkIjoiZWIwNjVmYWMtYTA1Ny00MzhiLWE2YWYtZjRlOTE1NzM4NmUyIn0sIndhcm5hZnRlciI6MTY4MDAxOTMxN30sIm5iZiI6MTY4MDAxNTcxMCwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmRlZmF1bHQ6ZGFzaHVzZXIifQ.L16iEbSmIKgY-FvaIR6dkZ-8wbo7bwtaRvo6ePx5KMizdnY-BN0jDDCqtZVMrmZqH2BTZIn-v67gZCbdXGLUHJU_u_jT8Yxh_D7gK2O1UbRm1pZE4hoKCXumrERSEMZaMJv6q-2A43VRvVpiw8el4glbLTvB-hQKN53ZSmJbBc4teWc-RUM5OF7yN-_HCjWId1cfWLmvJrEVzZ6AdYfe5q9IawEb6h7YoKWe-1QA1BYidEjOTvED8mD6elVNHznFEl9dh6OujdCCLL5oyICBTeThK2xn9eiTcEHcBcb3HL8_5ekC7OxA0zLRq-7vaaHdP-mzGWN92cZtuCi0B-4oFg

成功登陆

KubeConfig认证方式登录:
不使用 kubectl proxy
使用 kubectl apply 和 kubectl describe secret ... 及 grep 和剪切操作来为 default 服务帐户创建令牌，如下所示：

首先，创建 Secret，请求默认 ServiceAccount 的令牌：

kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: default-token
  annotations:
    kubernetes.io/service-account.name: default
type: kubernetes.io/service-account-token
EOF
接下来，等待令牌控制器使用令牌填充 Secret：

while ! kubectl describe secret default-token | grep -E '^token' >/dev/null; do
  echo "waiting for token..." >&2
  sleep 1
done
捕获并使用生成的令牌：

APISERVER=$(kubectl config view --minify | grep server | cut -f 2- -d ":" | tr -d " ")
TOKEN=$(kubectl describe secret default-token | grep -E '^token' | cut -f2 -d':' | tr -d " ")
curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure

输出类似于：

{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.1.149:443"
    }
  ]
}


kubectl craete serviceaccount dashuser
kubectl config set-cluster kubernetes --server="https://10.0.0.253:6443" --certificate-authority=/etc/kubernetes/pki/ca.crt --kubeconfig=./dashuser.conf

DEF_NS_ADMIN_TOKEN=$(kubectl get default-token -o jsonpath={.data.token} | base64 -d)

kubectl config set-credentials dashuser --token=$DEF_NS_ADMIN_TOKEN --kubeconfig=./dashuser.conf

kubectl config view --kubeconfig=./dashuser.conf

kubectl config use-context dashuser@kubernetes --kubeconfig=./dashuser.conf
```

Helm

```bash
#helm安装
下载对应包
#解压缩
[root@k8s-master01 ~]#tar -xvf helm-v3.11.2-linux-amd64.tar.gz
#移动
[root@k8s-master01 ~]#cp linux-amd64/helm /usr/local/bin/

#helm部署 面向生产可用的

#查看本地仓库
[root@k8s-master01 ~]#helm repo list

Bitnami很受欢迎的组织

https://artifacthub.io/
helm chart仓库

#helm搜索
#本地仓库
[root@k8s-master01 ~]#helm search repo mysql
NAME                    CHART VERSION   APP VERSION     DESCRIPTION                                       
bitnami/mysql           9.7.0           8.0.32          MySQL is a fast, reliable, scalable, and easy t...
bitnami/phpmyadmin      10.4.5          5.2.1           phpMyAdmin is a free software tool written in P...
bitnami/mariadb         11.5.5          10.6.12         MariaDB is an open source, community-developed ...
bitnami/mariadb-galera  7.5.4           10.6.12         MariaDB Galera is a multi-primary database clus...

#hub仓库  不需要打开artifacthub去搜索
[root@k8s-master01 ~]#helm search hub mysql
URL                                                     CHART VERSION   APP VERSION             DESCRIPTION                                       
https://artifacthub.io/packages/helm/cloudnativ...      5.0.1           8.0.16                  Chart to create a Highly available MySQL cluster  
https://artifacthub.io/packages/helm/ygqygq2/mysql      9.5.0           8.0.32                  MySQL is a fast, reliable, scalable, and easy t...
https://artifacthub.io/packages/helm/stakater/m...      1.0.6                                   mysql chart that runs on kubernetes               
https://artifacthub.io/packages/helm/kubesphere...      1.0.2           5.7.33                  High Availability MySQL Cluster, Open Source.     
https://artifacthub.io/packages/helm/bitnami/mysql      9.7.0           8.0.32                  MySQL is a fast, reliable, scalable, and easy t...
https://artifacthub.io/packages/helm/bitnami-ak...      9.4.4           8.0.31                  MySQL is a fast, reliable, scalable, and easy t...
https://artifacthub.io/packages/helm/kubegems/m...      8.9.6           8.0.29                  MySQL is a fast, reliable, scalable, and easy t...
………………………………………………………………………………………………省略


默认值文件定义
helm pull 拉取到本地
[root@k8s-master01 ~]#cd /data/
[root@k8s-master01 data]#helm pull bitnami/mysql
[root@k8s-master01 data]#ls
mysql-9.7.0.tgz  redis001  redis002
[root@k8s-master01 data]#tar xf mysql-9.7.0.tgz 
[root@k8s-master01 data]#ls
mysql  mysql-9.7.0.tgz  redis001  redis002
[root@k8s-master01 data]#vim mysql/templates/primary/statefulset.yaml

专门的相关开发人员去维护
bitnami经过大量生产环境的考验

默认文件改不了 先导出 再用helm install加载


#部署mysql
仅有主节点：
        helm install mysql  \
            --set auth.rootPassword=MageEdu \
            --set primary.persistence.storageClass=nfs-csi \
            --set auth.database=wpdb \
            --set auth.username=wpuser \
            --set auth.password='magedu.com' \
            bitnami/mysql \
            -n blog

        helm upgrade 
            release 更新 
        helm rollback
            release 回滚  

    带从节点：
        helm install mysql  \
            --set auth.rootPassword=MageEdu \
            --set global.storageClass=nfs-csi \
            --set architecture=replication \
            --set auth.database=wpdb \
            --set auth.username=wpuser \
            --set auth.password='magedu.com' \
            --set secondary.replicaCount=1 \
            --set auth.replicationPassword='replpass' \
            bitnami/mysql \
            -n blog

#报错
Error: INSTALLATION FAILED: create: failed to create: namespaces "blog" not found

#解决方案
[root@k8s-master01 data]#kubectl reate ns blog
创建blog ns即可


#部署成功提示
NAME: mysql
LAST DEPLOYED: Wed Mar 29 16:45:29 2023
NAMESPACE: blog
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: mysql
CHART VERSION: 9.7.0
APP VERSION: 8.0.32

** Please be patient while the chart is being deployed **

Tip:

  Watch the deployment status using the command: kubectl get pods -w --namespace blog

Services:

  echo Primary: mysql-primary.blog.svc.cluster.local:3306
  echo Secondary: mysql-secondary.blog.svc.cluster.local:3306

Execute the following to get the administrator credentials:

  echo Username: root
  MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace blog mysql -o jsonpath="{.data.mysql-root-password}" | base64 -d)

To connect to your database:

  1. Run a pod that you can use as a client:

      kubectl run mysql-client --rm --tty -i --restart='Never' --image  docker.io/bitnami/mysql:8.0.32-debian-11-r14 --namespace blog --env MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD --command -- bash

  2. To connect to primary service (read/write):

      mysql -h mysql-primary.blog.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"

  3. To connect to secondary service (read-only):

      mysql -h mysql-secondary.blog.svc.cluster.local -uroot -p"$MYSQL_ROOT_PASSWORD"

#卸载
helm uninstall mysql -n blog

#helm upgrade用来更新参数
helm upgrade mysql  \
            --set auth.rootPassword=MageEdu \
            --set global.storageClass=nfs-csi \
            --set architecture=replication \
            --set auth.database=wpdb \
            --set auth.username=wpuser \
            --set auth.password='magedu.com' \
            --set secondary.replicaCount=2 \
            --set auth.replicationPassword='replpass' \
            bitnami/mysql \
            -n blog

helm rollback 回滚

#部署wordpress
    Wordpress:
        自带的MariaDB:

            helm install wordpress \
                --set wordpressUsername=wpuser \
                --set wordpressPassword='magedu.com' \
                --set mariadb.auth.rootPassword=secretpassword \
                --set global.storageClass=nfs-csi \
                bitnami/wordpress

        外部的数据：
            helm install wordpress \
                --set mariadb.enabled=false \
                --set externalDatabase.host=mysql.blog.svc.cluster.local \
                --set externalDatabase.user=wpuser \
                --set externalDatabase.password='magedu.com' \
                --set externalDatabase.database=wpdb \
                --set externalDatabase.port=3306 \
                --set persistence.storageClass=nfs-csi \
                --set wordpressUsername=admin \
                --set wordpressPassword='magedu.com' \
                bitnami/wordpress \
                -n blog


        外部的数据，支持Ingress，且使用的mysql支持主从架构：
            helm install wordpress \
                --set mariadb.enabled=false \
                --set externalDatabase.host=mysql-primary.blog.svc.cluster.local \
                --set externalDatabase.user=wpuser \
                --set externalDatabase.password="magedu.com" \
                --set externalDatabase.database=wpdb \
                --set externalDatabase.port=3306 \
                --set persistence.storageClass=nfs-csi \
                --set ingress.enabled=true \
                --set ingress.ingressClassName=nginx \
                --set ingress.hostname=blog.magedu.com \
                --set ingress.pathType=Prefix \
                --set wordpressUsername=admin \
                --set wordpressPassword='magedu.com' \
                bitnami/wordpress \
                -n blog


#wordpress报错

连接不到数据库
[root@k8s-master01 ~]#kubectl get pod -n blog
NAME                         READY   STATUS    RESTARTS        AGE
mysql-primary-0              1/1     Running   0               167m
mysql-secondary-0            1/1     Running   0               167m
mysql-secondary-1            1/1     Running   0               160m
wordpress-599f54f6d6-prft4   1/1     Running   7 (6m55s ago)   27m

wordpress 13:32:08.93 
wordpress 13:32:08.93 Welcome to the Bitnami wordpress container
wordpress 13:32:08.94 Subscribe to project updates by watching https://github.com/bitnami/containers
wordpress 13:32:08.94 Submit issues and feature requests at https://github.com/bitnami/containers/issues
wordpress 13:32:08.94 
wordpress 13:32:08.94 INFO  ==> ** Starting WordPress setup **
realpath: /bitnami/apache/conf: No such file or directory
wordpress 13:32:08.98 INFO  ==> Configuring the HTTP port
wordpress 13:32:09.00 INFO  ==> Configuring the HTTPS port
wordpress 13:32:09.01 INFO  ==> Configuring Apache ServerTokens directive
wordpress 13:32:09.03 INFO  ==> Configuring PHP options
wordpress 13:32:09.04 INFO  ==> Setting PHP expose_php option
wordpress 13:32:09.06 INFO  ==> Validating settings in MYSQL_CLIENT_* env vars
wordpress 13:32:09.12 WARN  ==> You set the environment variable ALLOW_EMPTY_PASSWORD=yes. For safety reasons, do not use this flag in a production environment.
wordpress 13:32:09.23 INFO  ==> Ensuring WordPress directories exist
wordpress 13:32:09.24 INFO  ==> Trying to connect to the database server
wordpress 13:34:09.98 ERROR ==> Could not connect to the database

#解决方法
--set externalDatabase.password="magedu.com" \
密码实用双引号引起来

[root@k8s-master01 ~]#helm status mysql -n blog
查看部署信息

```







# error

从17.30到22.30

```bash
#wordpress报错

连接不到数据库

#解决方法
--set externalDatabase.password="magedu.com" \
密码实用双引号引起来

上述解决方案无效，单引号双引号不影响结果
根本问题在于apache重启，无法连接数据库
[root@k8s-master01 ~]#kubectl logs -f wordpress-577dd4b9b4-t46gq -n blog
wordpress 14:22:46.45 
wordpress 14:22:46.46 Welcome to the Bitnami wordpress container
wordpress 14:22:46.46 Subscribe to project updates by watching https://github.com/bitnami/containers
wordpress 14:22:46.46 Submit issues and feature requests at https://github.com/bitnami/containers/issues
wordpress 14:22:46.46 
wordpress 14:22:46.47 INFO  ==> ** Starting WordPress setup **
realpath: /bitnami/apache/conf: No such file or directory
wordpress 14:22:46.50 INFO  ==> Configuring the HTTP port
wordpress 14:22:46.51 INFO  ==> Configuring the HTTPS port
wordpress 14:22:46.52 INFO  ==> Configuring Apache ServerTokens directive
wordpress 14:22:46.54 INFO  ==> Configuring PHP options
wordpress 14:22:46.54 INFO  ==> Setting PHP expose_php option
wordpress 14:22:46.57 INFO  ==> Validating settings in MYSQL_CLIENT_* env vars
wordpress 14:22:56.63 WARN  ==> You set the environment variable ALLOW_EMPTY_PASSWORD=yes. For safety reasons, do not use this flag in a production environment.
wordpress 14:22:56.77 INFO  ==> Ensuring WordPress directories exist
wordpress 14:22:56.79 INFO  ==> Trying to connect to the database server
wordpress 14:23:04.83 INFO  ==> Configuring WordPress with settings provided via environment variables
wordpress 14:23:06.16 INFO  ==> Installing WordPress
wordpress 14:23:48.30 INFO  ==> Persisting WordPress installation
wordpress 14:24:42.53 INFO  ==> ** WordPress setup finished! **

wordpress 14:24:42.58 INFO  ==> ** Starting Apache **
[Wed Mar 29 14:24:42.758907 2023] [ssl:warn] [pid 1] AH01909: www.example.com:8443:0 server certificate does NOT include an ID which matches the server name
[Wed Mar 29 14:24:42.759683 2023] [ssl:warn] [pid 1] AH01909: www.example.com:443:0 server certificate does NOT include an ID which matches the server name
[Wed Mar 29 14:24:42.862941 2023] [ssl:warn] [pid 1] AH01909: www.example.com:8443:0 server certificate does NOT include an ID which matches the server name
[Wed Mar 29 14:24:42.863854 2023] [ssl:warn] [pid 1] AH01909: www.example.com:443:0 server certificate does NOT include an ID which matches the server name
[Wed Mar 29 14:24:42.961488 2023] [mpm_prefork:notice] [pid 1] AH00163: Apache/2.4.56 (Unix) OpenSSL/1.1.1n configured -- resuming normal operations
[Wed Mar 29 14:24:42.962227 2023] [core:notice] [pid 1] AH00094: Command line: '/opt/bitnami/apache/bin/httpd -f /opt/bitnami/apache/conf/httpd.conf -D FOREGROUND'
10.0.0.111 - - [29/Mar/2023:14:24:45 +0000] "GET /wp-login.php HTTP/1.1" 200 5157
192.168.194.28 - - [29/Mar/2023:14:24:53 +0000] "POST /wp-cron.php?doing_wp_cron=1680099893.8016209602355957031250 HTTP/1.1" 200 -
10.0.0.111 - - [29/Mar/2023:14:24:55 +0000] "GET /wp-login.php HTTP/1.1" 200 5157
10.0.0.111 - - [29/Mar/2023:14:24:55 +0000] "GET /wp-admin/install.php HTTP/1.1" 200 1232
10.0.0.111 - - [29/Mar/2023:14:25:05 +0000] "GET /wp-login.php HTTP/1.1" 200 5157
10.0.0.111 - - [29/Mar/2023:14:25:05 +0000] "GET /wp-admin/install.php HTTP/1.1" 200 1232
10.0.0.111 - - [29/Mar/2023:14:25:10 +0000] "GET /wp-login.php HTTP/1.1" 200 5157
10.0.0.111 - - [29/Mar/2023:14:25:15 +0000] "GET /wp-admin/install.php HTTP/1.1" 200 1232
10.0.0.111 - - [29/Mar/2023:14:25:15 +0000] "GET /wp-login.php HTTP/1.1" 200 5157
10.0.0.111 - - [29/Mar/2023:14:25:25 +0000] "GET /wp-login.php HTTP/1.1" 200 5157
10.0.0.111 - - [29/Mar/2023:14:25:25 +0000] "GET /wp-admin/install.php HTTP/1.1" 200 1232
192.168.194.28 - - [29/Mar/2023:14:25:33 +0000] "POST /wp-cron.php?doing_wp_cron=1680099933.2975089550018310546875 HTTP/1.1" 200 -
192.168.194.28 - - [29/Mar/2023:14:25:33 +0000] "POST /wp-cron.php?doing_wp_cron=1680099933.3434059619903564453125 HTTP/1.1" 200 -
10.0.0.111 - - [29/Mar/2023:14:25:35 +0000] "GET /wp-login.php HTTP/1.1" 200 5157
10.0.0.111 - - [29/Mar/2023:14:25:35 +0000] "GET /wp-admin/install.php HTTP/1.1" 200 1232
[Wed Mar 29 14:25:50.412764 2023] [mpm_prefork:notice] [pid 1] AH00169: caught SIGTERM, shutting down
```

