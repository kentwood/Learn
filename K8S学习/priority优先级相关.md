1. 获取与描述priorityclass
```
kubectl get priorityclass
```
```
kubectl describe priorityclass system-node-critical
```
以下是一个PriorityClass的yaml
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "This priority class is used for high-priority pods."
preemptionPolicy: PreemptLowerPriority
```

2. 关于priorityclass的抢占策略preemptionPolicy
- PreemptLowerPriority（默认）
  - 允许抢占低优先级Pod
  - 适用于关键业务Pod
  - 保证高优先级Pod能够调度成功
- Never
  - 永远不抢占其他Pod
  - 适用于批处理任务或对稳定性要求高的场景
  - 只能等待资源自然释放


3. 关于globalDefault
- 作用：为没有明确指定 priorityClassName 的 Pod 提供默认优先级
- 唯一性：整个集群中只能有一个 PriorityClass 设置 globalDefault: true
- 自动应用：新创建的 Pod 如果没有指定优先级类，会自动使用这个默认设置

4. 具体使用
创建一个pod，使用对应的PriorityClass
```yaml
# low-prio-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: low-prio-pod
spec:
  containers:
  - name: nginx
    image: nginx
  priorityClassName: low-priority
```