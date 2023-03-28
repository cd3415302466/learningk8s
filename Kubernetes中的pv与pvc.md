# Kubernetes中的pv与pvc

准备

nfs服务器安装

```shell
apt install nfs-kernel-server -y

vim /etc/exportfs
# /etc/exports: the access control list for filesystems which may be exported
#       to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#

/data/redis001 10.0.0.0/16(rw,no_subtree_check,no_root_squash)
/data/redis002 10.0.0.0/16(rw,no_subtree_check,no_root_squash)



exportfs -ar

showmount -e 10.0.0.104
exportfs
```

k8s控制平面各个节点

```bash
apt install nfs-common -y

安装csi-nfs-driver
```

pv与pvc的概念

```shell
    定义存储：由管理完成
    消费存储：最终用户
        PV/PVC 
            pv: persistent volume 
                标准的API资源类型，支持声明式创建
                由管理员负责管理
                集群级别的资源 
                不能够被Pod直接使用

            pvc: persistent volume claim 
                标准的API资源类型，支持声明式创建
                由最终用户负责创建
                名称空间级别的资源 
                可被Pod直接使用
                    In-Tree的卷插件类型
                        persistentVolumeClaim:
                            claimName: <PVC_NAME>

        PVC <-- 1V1 --> PV 
            绑定

        PV如何满足PVC期望？
            1、管理员事先创建多种规格的PV； 
            2、管理流程：提交工单；

                PV静态置备（Static Provisioning）
            
            3、管理员不去创建PV，取而代之，创建StorageClass（PV模板）
                管理员创建SC资源对象：配置了远程存储服务管理接口的API调用操作

                远程存储服务：有管理API，而且还要支持远程访问

                PV动态置备（Dynamic Provisioning）
```

