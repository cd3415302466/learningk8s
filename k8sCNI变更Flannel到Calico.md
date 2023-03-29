```
kubeadm reset --cri-socket unix:///run/cri-dockerd.sock
rm -rf /etc/kubernetes/ /var/lib/kubelet /var/lib/dockershim /var/run/kubernetes /var/lib/cni /etc/cni/net.d $HOME/.kube



kubeadm reset  && systemctl stop kubelet && rm -rf /etc/kubernetes/ && rm -rf ~/.kube/ && rm -rf /var/lib/etcd/
```

k8s拆集群指定unix

```
kubeadm reset --cri-socket unix:///var/run/cri-dockerd.sock
```

k8s calico插件

```bash
#初始化k8s控制平面
kubeadm init --control-plane-endpoint="kubeapi.magedu.com" --kubernetes-version=v1.26.2 --pod-network-cidr=192.168.0.0/16 --service-cidr=10.96.0.0/12 --token-ttl=0 --cri-socket unix:///run/cri-dockerd.sock --upload-certs --image-repository=registry.aliyuncs.com/google_containers

#calico插件
#下载
curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico-etcd.yaml -O

curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml -O

vim calico.yaml
            - name: CALICO_IPV4POOL_IPIP
              value: "Always"
              # value: "Cross-Subnet"                #使用ipip还是BGP
            - name: CALICO_IPV4POOL_CIDR
              value: "192.168.0.0/16"
            - name: CALICO_IPV4POOL_BLOCK_SIZE
              value: "24"
              
kubectl apply -f calico.yaml

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join kubeapi.magedu.com:6443 --token 32hb2t.pneh4vhqtmqpqop5 \
        --discovery-token-ca-cert-hash sha256:58e44e8a4d8e189ade5ecd67640cde1f30f2f40f1ae22bacfe0aa3695846aa33 \
        --control-plane --certificate-key d3e9f2a0aa34cbbaa399d203624454111f2ec4445aae96d9e8bcba079abb8652 --cri-socket unix:///run/cri-dockerd.sock

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join kubeapi.magedu.com:6443 --token 32hb2t.pneh4vhqtmqpqop5 \
        --discovery-token-ca-cert-hash sha256:58e44e8a4d8e189ade5ecd67640cde1f30f2f40f1ae22bacfe0aa3695846aa33 --cri-socket unix:///run/cri-dockerd.sock
```

添加pod

```bash
create deployment demoapp --image ikubernetes/demoapp:v1.0 --replicas=6
#添加测试pod
kubectl run client-$RANDOM --image ikubernetes/admin-box:v1.2 --restart=Never -it --command -- /bin/sh
```

变更网络插件后

```bash
#IPIP模式
[root@k8s-node01 ~]#route -n
内核 IP 路由表
目标            网关            子网掩码        标志  跃点   引用  使用 接口
0.0.0.0         10.0.0.2        0.0.0.0         UG    0      0        0 eth0
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.11.0    10.0.0.102      255.255.255.0   UG    0      0        0 tunl0
192.168.70.0    10.0.0.113      255.255.255.0   UG    0      0        0 tunl0
192.168.87.0    10.0.0.101      255.255.255.0   UG    0      0        0 tunl0
192.168.194.0   0.0.0.0         255.255.255.0   U     0      0        0 *
192.168.194.3   0.0.0.0         255.255.255.255 UH    0      0        0 calib15f9a66a9e
192.168.194.4   0.0.0.0         255.255.255.255 UH    0      0        0 calia7dc9644370
192.168.194.6   0.0.0.0         255.255.255.255 UH    0      0        0 cali0b9a75b18e7
192.168.252.0   10.0.0.112      255.255.255.0   UG    0      0        0 tunl0

[root@k8s-node01 ~]#ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:c6:ac:f9 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.111/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fec6:acf9/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:dd:04:53:fb brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1480 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
    inet 192.168.194.0/32 scope global tunl0
       valid_lft forever preferred_lft forever
9: calib15f9a66a9e@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
10: calia7dc9644370@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
14: cali0b9a75b18e7@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever

#ipip模式下抓包

tcpdump -i eth0 -nn host 10.0.0.112

19:29:00.193688 IP 10.0.0.111 > 10.0.0.112: IP 192.168.194.6.51130 > 192.168.252.3.80: Flags [S], seq 1459633433, win 64800, options [mss 1440,sackOK,TS val 3852180261 ecr 0,nop,wscale 7], length 0 (ipip-proto-4)
19:29:00.194106 IP 10.0.0.112 > 10.0.0.111: IP 192.168.252.3.80 > 192.168.194.6.51130: Flags [S.], seq 3206594998, ack 1459633434, win 64260, options [mss 1440,sackOK,TS val 467187100 ecr 3852180261,nop,wscale 7], length 0 (ipip-proto-4)
19:29:00.194154 IP 10.0.0.111 > 10.0.0.112: IP 192.168.194.6.51130 > 192.168.252.3.80: Flags [.], ack 1, win 507, options [nop,nop,TS val 3852180261 ecr 467187100], length 0 (ipip-proto-4)
19:29:00.194493 IP 10.0.0.111 > 10.0.0.112: IP 192.168.194.6.51130 > 192.168.252.3.80: Flags [P.], seq 1:78, ack 1, win 507, options [nop,nop,TS val 3852180262 ecr 467187100], length 77: HTTP: GET / HTTP/1.1 (ipip-proto-4)
19:29:00.194749 IP 10.0.0.112 > 10.0.0.111: IP 192.168.252.3.80 > 192.168.194.6.51130: Flags [.], ack 78, win 502, options [nop,nop,TS val 467187100 ecr 3852180262], length 0 (ipip-proto-4)
19:29:00.196306 IP 10.0.0.112 > 10.0.0.111: IP 192.168.252.3.80 > 192.168.194.6.51130: Flags [P.], seq 1:18, ack 78, win 502, options [nop,nop,TS val 467187102 ecr 3852180262], length 17: HTTP: HTTP/1.0 200 OK (ipip-proto-4)
19:29:00.196374 IP 10.0.0.111 > 10.0.0.112: IP 192.168.194.6.51130 > 192.168.252.3.80: Flags [.], ack 18, win 507, options [nop,nop,TS val 3852180264 ecr 467187102], length 0 (ipip-proto-4)
19:29:00.196641 IP 10.0.0.112 > 10.0.0.111: IP 192.168.252.3.80 > 192.168.194.6.51130: Flags [FP.], seq 18:270, ack 78, win 502, options [nop,nop,TS val 467187102 ecr 3852180262], length 252: HTTP (ipip-proto-4)
19:29:00.197030 IP 10.0.0.111 > 10.0.0.112: IP 192.168.194.6.51130 > 192.168.252.3.80: Flags [F.], seq 78, ack 271, win 506, options [nop,nop,TS val 3852180264 ecr 467187102], length 0 (ipip-proto-4)
19:29:00.197316 IP 10.0.0.112 > 10.0.0.111: IP 192.168.252.3.80 > 192.168.194.6.51130: Flags [.], ack 79, win 502, options [nop,nop,TS val 467187103 ecr 3852180264], length 0 (ipip-proto-4)

#如何更改ipip到cross-subenet
打开配置文件 
vim calico.yaml
将Always注释掉 启用Across-Subnet

kubectl get ippool default-ipv4-ippool -o yaml
kubectl get ippool default-ipv4-ippool -o yaml > default-ipv4-ippool.yaml
vim default-ipv4-ippool.yaml 

将ipipMode: Always 改为ipipMode: CrossSubnet

kubectl apply -f default-ipv4-ippool.yaml 
kubectl get ippool default-ipv4-ippool -o yaml

#修改失败的话 将原有的 ippool删掉 k8s集群系统会自动创建新的acrosssubnet ippool
kubectl delete -f default-ipv4-ippool.yaml

[root@k8s-node01 ~]#route -n
内核 IP 路由表
目标            网关            子网掩码        标志  跃点   引用  使用 接口
0.0.0.0         10.0.0.2        0.0.0.0         UG    0      0        0 eth0
10.0.0.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.11.0    10.0.0.102      255.255.255.0   UG    0      0        0 eth0
192.168.70.0    10.0.0.113      255.255.255.0   UG    0      0        0 eth0
192.168.87.0    10.0.0.101      255.255.255.0   UG    0      0        0 eth0
192.168.194.0   0.0.0.0         255.255.255.0   U     0      0        0 *
192.168.194.3   0.0.0.0         255.255.255.255 UH    0      0        0 calib15f9a66a9e
192.168.194.4   0.0.0.0         255.255.255.255 UH    0      0        0 calia7dc9644370
192.168.252.0   10.0.0.112      255.255.255.0   UG    0      0        0 eth0

192.168.194.3 192.168.194.4 通过calia7dc9644370发送  IPIP二者MTU均为 mtu 1480


[root@k8s-node01 ~]#ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:c6:ac:f9 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.111/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fec6:acf9/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:dd:04:53:fb brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1480 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
    inet 192.168.194.0/32 scope global tunl0
       valid_lft forever preferred_lft forever
9: calib15f9a66a9e@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
10: calia7dc9644370@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1480 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
       
#Across—Subnet模式下抓包
[root@k8s-node01 ~]#tcpdump -i eth0 -nn tcp port 80

19:44:12.474196 IP 192.168.194.6.55606 > 192.168.252.3.80: Flags [S], seq 2085634331, win 64800, options [mss 1440,sackOK,TS val 3853092541 ecr 0,nop,wscale 7], length 0
19:44:12.474634 IP 192.168.252.3.80 > 192.168.194.6.55606: Flags [S.], seq 507089807, ack 2085634332, win 64260, options [mss 1440,sackOK,TS val 468099380 ecr 3853092541,nop,wscale 7], length 0
19:44:12.474692 IP 192.168.194.6.55606 > 192.168.252.3.80: Flags [.], ack 1, win 507, options [nop,nop,TS val 3853092542 ecr 468099380], length 0
19:44:12.474951 IP 192.168.194.6.55606 > 192.168.252.3.80: Flags [P.], seq 1:78, ack 1, win 507, options [nop,nop,TS val 3853092542 ecr 468099380], length 77: HTTP: GET / HTTP/1.1
19:44:12.475155 IP 192.168.252.3.80 > 192.168.194.6.55606: Flags [.], ack 78, win 502, options [nop,nop,TS val 468099381 ecr 3853092542], length 0
19:44:12.477084 IP 192.168.252.3.80 > 192.168.194.6.55606: Flags [P.], seq 1:18, ack 78, win 502, options [nop,nop,TS val 468099382 ecr 3853092542], length 17: HTTP: HTTP/1.0 200 OK
19:44:12.477533 IP 192.168.194.6.55606 > 192.168.252.3.80: Flags [.], ack 18, win 507, options [nop,nop,TS val 3853092544 ecr 468099382], length 0
19:44:12.477665 IP 192.168.252.3.80 > 192.168.194.6.55606: Flags [FP.], seq 18:270, ack 78, win 502, options [nop,nop,TS val 468099383 ecr 3853092542], length 252: HTTP
19:44:12.478139 IP 192.168.194.6.55606 > 192.168.252.3.80: Flags [F.], seq 78, ack 271, win 506, options [nop,nop,TS val 3853092545 ecr 468099383], length 0
19:44:12.478749 IP 192.168.252.3.80 > 192.168.194.6.55606: Flags [.], ack 79, win 502, options [nop,nop,TS val 468099384 ecr 3853092545], length 0

因为是基于BGP直接走路由 所以不会再出现ip封装模式下10.0.0.111和10.0.0.112的端口
非同一个网络通过BGP学习对端网段的路由表 通过查路由表 去访问别的pod

flannel为通过flanneld注册中心获取而来
```

工具calicoctl的使用 （管理calico）

```bash
#下载
curl -L https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64 -o calicoctl

[root@k8s-master01 ~]#cp calicoctl /usr/local/bin/
[root@k8s-master01 ~]#chmod +x /usr/local/bin/calicoctl
[root@k8s-master01 ~]#ll /usr/local/bin/calicoctl
-rwxr-xr-x 1 root root 59594985 3月  28 19:51 /usr/local/bin/calicoctl*

[root@k8s-master01 ~]#calicoctl --help
Usage:
  calicoctl [options] <command> [<args>...]

    create       Create a resource by file, directory or stdin.
    replace      Replace a resource by file, directory or stdin.
    apply        Apply a resource by file, directory or stdin.  This creates a resource
                 if it does not exist, and replaces a resource if it does exists.
    patch        Patch a pre-exisiting resource in place.
    delete       Delete a resource identified by file, directory, stdin or resource type and
                 name.
    get          Get a resource identified by file, directory, stdin or resource type and
                 name.
    label        Add or update labels of resources.
    convert      Convert config files between different API versions.
    ipam         IP address management.
    node         Calico node management.
    version      Display the version of this binary.
    datastore    Calico datastore management.

Options:
  -h --help                    Show this screen.
  -l --log-level=<level>       Set the log level (one of panic, fatal, error,
                               warn, info, debug) [default: panic]
     --context=<context>       The name of the kubeconfig context to use.
     --allow-version-mismatch  Allow client and cluster versions mismatch.

Description:
  The calicoctl command line tool is used to manage Calico network and security
  policy, to view and manage endpoint configuration, and to manage a Calico
  node instance.

  See 'calicoctl <command> --help' to read about a specific subcommand.
  
[root@k8s-master01 ~]#calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.0.0.102   | node-to-node mesh | up    | 11:39:42 | Established |
| 10.0.0.111   | node-to-node mesh | up    | 11:39:41 | Established |
| 10.0.0.112   | node-to-node mesh | up    | 11:39:41 | Established |
| 10.0.0.113   | node-to-node mesh | up    | 11:39:42 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

node-to-node mesh 点到点的网格模式
BGP路由协议的学习，学习对端路由表

什么是CRD？
在Kubernetes中，CRD是自定义资源定义的缩写。它们允许用户扩展Kubernetes API，并向集群添加自定义资源类型。可以使用CRD来创建以前未在Kubernetes中定义的新资源类型，或者扩展现有资源类型的属性。

通过创建CRD，您可以定义一个新的API端点，该端点支持使用自定义对象的Kubernetes API操作（例如，创建、读取、更新和删除）。这使得您可以将自定义业务逻辑与Kubernetes集群紧密集成。

在使用CRD时，需要定义每个资源的名称、版本、属性和行为。一旦定义了CRD，您就可以使用kubectl命令行工具或Kubernetes API与自定义资源进行交互。

总之，CRD是一种强大的机制，可让您将自定义资源添加到Kubernetes集群中，并且这些资源可以像内置资源一样使用。

[root@k8s-master01 ~]#kubectl get crds （自定义资源类型）
NAME                                                  CREATED AT
bgpconfigurations.crd.projectcalico.org               2023-03-28T10:51:24Z
bgppeers.crd.projectcalico.org                        2023-03-28T10:51:24Z
blockaffinities.crd.projectcalico.org                 2023-03-28T10:51:24Z
caliconodestatuses.crd.projectcalico.org              2023-03-28T10:51:24Z
clusterinformations.crd.projectcalico.org             2023-03-28T10:51:24Z
felixconfigurations.crd.projectcalico.org             2023-03-28T10:51:24Z
globalnetworkpolicies.crd.projectcalico.org           2023-03-28T10:51:24Z
globalnetworksets.crd.projectcalico.org               2023-03-28T10:51:24Z
hostendpoints.crd.projectcalico.org                   2023-03-28T10:51:24Z
ipamblocks.crd.projectcalico.org                      2023-03-28T10:51:24Z
ipamconfigs.crd.projectcalico.org                     2023-03-28T10:51:24Z
ipamhandles.crd.projectcalico.org                     2023-03-28T10:51:24Z
ippools.crd.projectcalico.org                         2023-03-28T10:51:24Z
ipreservations.crd.projectcalico.org                  2023-03-28T10:51:24Z
kubecontrollersconfigurations.crd.projectcalico.org   2023-03-28T10:51:24Z
networkpolicies.crd.projectcalico.org                 2023-03-28T10:51:24Z
networksets.crd.projectcalico.org                     2023-03-28T10:51:24Z
```







报错

```bash
Failed to create pod sandbox: rpc error: code = Unknown desc = [failed to set up sandbox container "56ba67e344a969f8546aebb45cb825045727ecb3bc5074855e4ec10b98472136" network for pod "coredns-5bbd96d687-drpxn": networkPlugin cni failed to set up pod "coredns-5bbd96d687-drpxn_kube-system" network: error getting ClusterInformation: Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes"), failed to clean up sandbox container "56ba67e344a969f8546aebb45cb825045727ecb3bc5074855e4ec10b98472136" network for pod "coredns-5bbd96d687-drpxn": networkPlugin cni failed to teardown pod "coredns-5bbd96d687-drpxn_kube-system" network: error getting ClusterInformation: Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")]
```

解决方法

```bash
            - name: CLUSTER_TYPE
              value: "k8s,bgp"
              #下方新增
            - name: IP_AUTODETECTION_METHOD
              value: "interface=eth0"
              
解决无效
```

报错

```bash
error: error parsing calico-etcd.yaml: error converting YAML to JSON: yaml: line 199: did not find expected '-' indicator
```

报错

```shell
Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")
```

解决方法

```shell
拆集群时
rm -rf $HOME/.kube 命令删除这个目录

再执行
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

