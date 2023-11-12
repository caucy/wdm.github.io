# 常见问题定位方式

## 如何用kubectl如何连上apiserver
找到kubeconfig，指定Kubeconfig 后，可以连上pod
```
kubectl exec pod_name bash -it # 可以连上pod，默认使用~/.kube/config
```

## pod pending 如何定位
```
kubectl describe pod_name #可以看到pod 状态，以及pod 异常原因
```

## pod 进程oom 如何定位
```
dmesg -T 可以看oom 事件
```

## 从本地拷贝文件到pod，或者反过来拷贝
```
kubectl cp pod_name:/dir_/file_name file_name # pod 拷贝到本地
```

## 查看所有的pod, nodes， configmap 等
```
kubectl pods/nodes/configmaps ...
```

## 查看nodelevel 相关宿主资源是否缺失
```
kubectl get node -L nodeLevel
```
