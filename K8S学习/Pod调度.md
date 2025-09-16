# scheduling

### 1、手动分配pod到指定的节点，使用nodeName字段
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: node01
  containers:
  -  image: nginx
     name: nginx
```
这种情况下不需要scheduler参与

另外，也可以使用node的标签来分类pod到node，但是使用nodeSelector标签需要scheduler参与
```yaml
pods/pod-nginx.yaml
Copy pods/pod-nginx.yaml to clipboard
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

### 2、使用标签查询资源
```
kubectl get pods --selector env=dev
```
使用标签查询pod并且统计个数
```
kubectl get pods --selector env=dev --no-headers | wc -l
```
多个标签查询用逗号隔开
```
kubectl get pods --selector bu=finance,env=prod,tier=frontend
```
### 3、Taints and Tolerations
为节点创建Taints
```
kubectl taint nodes node01 spray=mortein:NoSchedule 
```
taints的格式 key=value:effect （taints添加时可以没有value，但是一定要有key和effect）


effect有三种模式：

| Effect | 作用 | 对正在运行pod的影响 |
|-------|-------|-------|
| NoSchedule | 禁止调度：不容忍该污点的 Pod 不会被调度到节点 | 不影响已存在的 Pod |
| PreferNoSchedule | 尽量避免调度：尽量不调度不容忍的 Pod，但如果没有其他节点可用时仍会调度 | 不影响已存在的 Pod |
| NoExecute | 禁止调度 + 驱逐：不容忍的 Pod 不会被调度，且已运行的 Pod 会被驱逐 | 驱逐不容忍的 Pod |

如果要让pod能在加了taint为 **spray=mortein:NoSchedule** 的节点上运行，需要为pod设置如下的tolerations

```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: bee
spec:
  containers:
  - image: nginx
    name: bee
  tolerations:
  - key: spray
    value: mortein
    effect: NoSchedule
    operator: Equal
```

删除taints
```
kubectl taint nodes node1 spray=mortein:NoSchedule-
```
或者直接删除taints的key也可
```
kubectl taint nodes node1 spray-
```


