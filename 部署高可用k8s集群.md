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

kubectl apply -f deploy/v3.1.0/

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

metrics-server报错

```bash

0328 22:06:01.248567  302307 memcache.go:255] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0328 22:06:01.265268  302307 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0328 22:06:01.268921  302307 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
E0328 22:06:01.272115  302307 memcache.go:106] couldn't get resource list for metrics.k8s.io/v1beta1: the server is currently unable to handle the request
```

解决方法

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

