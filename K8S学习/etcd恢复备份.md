# 备份与恢复

## 备份etcd
1、查看etcd版本，可以直接describe ectd的pod来查看
```
kubectl describe pod etcd-controlplane -n kube-system
```

2、etcd的访问地址，查看etcd pod里的--listen-client-urls
```
https://127.0.0.1:2379
```
3、查看etcd的server的密钥信息，主要查看以下三个变量
```
--cert-file=/etc/kubernetes/pki/etcd/server.crt
--key-file=/etc/kubernetes/pki/etcd/server.key
--trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

4、使用etcd命令行备份etcd数据，保存到/opt/snapshot-pre-boot.db里
```
etcdctl --endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot save /opt/snapshot-pre-boot.db
```

## 恢复数据
1、执行etcd命令行，将db数据写入
etcdutl snapshot restore /opt/snapshot-pre-boot.db --data-dir /var/lib/etcd-from-backup
（注意这个写入的backup目录和原来数据的目录不一致，确保数据不混杂起来）

2、修改etcd的yaml，在/etc/kubernetes/manifests/etcd.yaml，修改hostPath的路径
```yaml
  volumes:
  - hostPath:
      path: /var/lib/etcd-from-backup # Newly restored backup directory
      type: DirectoryOrCreate
    name: etcd-data
```
