# k8s常用操作命令







```bash
仅打印资源清单
kubectl create deployment nginx --image=nginx:1.22-alpine --replicas=1 --dry-run=client -o yaml
```

