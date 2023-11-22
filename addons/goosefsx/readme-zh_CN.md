# GoosefsX

该插件用于 GoosefsX

## 安装

```shell
kubectl apply -f runtime-profile.yaml
```

## 使用

### 创建 Dataset 和 ThinRuntime 
```shell
$ cat <<EOF > dataset.yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: goosefsx-demo
spec:
  mounts:
  - mountPoint: <cos>
    name: goosefsx-demo
---
apiVersion: data.fluid.io/v1alpha1
kind: ThinRuntime
metadata:
  name: goosefsx-demo
spec:
  profileName: goosefsx
  fuse:
    nodeSelector:
      fluid: thin
EOF

$ kubectl apply -f dataset.yaml
```
修改上面的 mountPoint，指向要使用的 cos 存储桶
修改上面的 fuse.nodeSelector

### 运行Pod，并且使用Fluid PVC

```shell
$ cat <<EOF > app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: goosefsx-demo
  volumes:
    - name: goosefsx-demo
      persistentVolumeClaim:
        claimName: goosefsx-demo
EOF

$ kubectl apply -f app.yaml
```
使用远程文件系统的应用部署完成后，对应的FUSE pod也会被调度到同一个节点上。

```shell
$ kubectl get pods
NAME                  READY   STATUS    RESTARTS   AGE
goosefsx-demo-fuse-wx7ns   1/1     Running   0          12s
nginx                 1/1     Running   0          26s
```
The remote file system is mounted to the /data directory of nginx pod.


## TODO
