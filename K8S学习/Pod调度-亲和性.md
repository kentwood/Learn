# 接pod调度单独开一个文档

### 4、Node Affinity
为节点添加标签
```
kubectl label node node01 color=blue
```

创建deployment时设置nodeAffinity
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  template:
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        imagePullPolicy: Always
        name: nginx
      affinity:            # affinity rules added from here
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: color
                operator: In
                values:
                - blue
```
关于nodeAffinity的类型和作用：

### 1、硬亲和性 (requiredDuringSchedulingIgnoredDuringExecution)
作用：必须满足的条件，否则 Pod 无法调度

特点：

调度时强制要求

不满足条件时 Pod 保持 Pending 状态

节点标签变更不会影响已运行 Pod

### 2、软亲和性 (preferredDuringSchedulingIgnoredDuringExecution)
作用：优先满足的条件，但不强制

特点：

调度器尝试满足但不保证

不满足时仍可调度到其他节点

支持权重设置（1-100）

|操作符	|说明	|适用场景|
|-------|-------|-------|
|In	|标签值在指定列表中	|匹配多个可选值|
|NotIn	|标签值不在指定列表中	|排除特定值|
|Exists	|标签键存在（不检查值）	|只需标签存在|
|DoesNotExist|	标签键不存在	排除特定标签
|Gt (Greater)|	标签值大于指定值（整数）|	资源比较（CPU/内存）|
|Lt (Less)|	标签值小于指定值（整数）	|资源比较（CPU/内存）|
