# k8s学习笔记

k8s错误

failed pulling image \"registry.k8s.io/pause:3.6\ pause3.6镜像拉取失败

反复失败的原因 k8s pause:3.6镜像拉取失败

kubeadm安装 从入门到放弃

关键

```
pause:3.6镜像
拖入到虚拟机
运行 docker load -i pause-3.6.tar 进行镜像导入
不然kubelet会一直从registry.k8s.io/pause:3.6 拉取
```





实操 wordpress

```shell
github https://github.com/iKubernetes/learning-k8s

git clone https://github.com/iKubernetes/learning-k8s.git

[root@k8s-master01 ~]#ls
1996  cri-dockerd_0.3.0.3-0.ubuntu-focal_amd64.deb  learning-k8s  pause-3.6.tar  snap  vim
[root@k8s-master01 ~]#cd learning-k8s/
[root@k8s-master01 learning-k8s]#ls
csi-driver-nfs  examples  helm                 jenkins  LICENSE         OpenELB    tutorials
dashboard       gitlab    ingress-canary-demo  Kuboard  metrics-server  README.md  wordpress
[root@k8s-master01 learning-k8s]#cd wordpress/
[root@k8s-master01 wordpress]#ls
mysql  mysql-ephemeral  nginx  README.md  wordpress  wordpress-apache-ephemeral

kubectl apply -f mysql-ephemeral/
```

