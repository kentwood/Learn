# 升级k8s集群

1、查看当前版本
```
kubectl get node 
```
可查看集群所有节点的k8s版本  
升级策略采用一个节点一个节点的升级，每次升级一个小版本

2、查看可升级的版本
```
kubeadm upgrade plan
```
kubeadm本身也有版本，只能升级到kubeadm版本支持的k8s版本

3、先drain一个node，排除pod与标记为不可分配
```
kubectl drain node controlplane --ignore-daemonsets
```
4、更新kubeadm的源，以升级1.32到1.33为例
```
vim /etc/apt/sources.list.d/kubernetes.list
```
修改为以下内容
```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /
```
5、执行
```
apt update
```
更新apt源信息

6、列出可用的kubeadm版本
```
apt-cache madison kubeadm
```
7、根据步骤6得到的kubeadm版本进行安装
```
apt-get install kubeadm=1.33.0-1.1
```
8、升级集群
```
kubeadm upgrade apply v1.33.0
```
9、升级kubelet
```
apt-get install kubelet=1.33.0-1.1
```
10、重启kubelet
```
systemctl daemon-reload
systemctl restart kubelet
```
11、需要对节点进行uncordon
```
kubectl uncordon controlplane
```
12、对其他节点进行drain
```
kubeclt drain node01 --ignore-daemonsets
```
13、ssh到其他节点
```
ssh node01
```
14、更新kubeadm的源，以升级1.32到1.33为例
```
vim /etc/apt/sources.list.d/kubernetes.list
```
修改为以下内容
```
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /
```
15、执行
```
apt update
```
更新apt源信息

16、列出可用的kubeadm版本
```
apt-cache madison kubeadm
```
17、根据步骤16得到的kubeadm版本进行安装
```
apt-get install kubeadm=1.33.0-1.1
```
18、升级node，这里和步骤8主节点的升级不一样，执行以下命令即可
```
kubeadm upgrade node
```
19、升级kubelet
```
apt-get install kubelet=1.33.0-1.1
```
20、重启kubelet
```
systemctl daemon-reload
systemctl restart kubelet
```
21、exit退回主节点
```
exit
```
22、恢复node为可分配
```
kubectl uncordon node01
```