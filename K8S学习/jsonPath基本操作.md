1. 将查询到的所有信息通过json形式输出
```
kubectl get nodes -o json
```

2. 使用jsonpath获取所有node的名字
```
kubectl get node -o jsonpath='{.items[*].metadata.name}'
```
输出结果为
```
controlplane node01
```

3. 使用jsonpath查看kubeconfig
```
kubectl config view --kubeconfig=my-kube-config  -o jsonpath="{.users[*].name}"
```

4. 根据pv的容量排序输出
先查pv的json输出
```
kubectl get pv -o json
```
查到items的数据结构
```
kubectl get pv --sort-by=.spec.capacity.storage
```

5. 排序与获取特定字段信息
```
kubectl get pv --sort-by=.spec.capacity.storage -o=custom-columns=NAME:.metadata.name,CAPACITY:.spec.capacity.storage
```

6. 精确查询
查询config下contexts数组中，.context.user=aws-user的name
```
kubectl config view --kubeconfig=my-kube-config -o jsonpath="{.contexts[?(@.context.user=='aws-user')].name}"
```