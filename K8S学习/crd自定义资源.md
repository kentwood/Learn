## 基本概念
在 Kubernetes（K8s）中，CRD（CustomResourceDefinition，自定义资源定义） 是一种扩展 Kubernetes API 的机制，允许用户创建自定义资源（Custom Resource，CR），从而将 Kubernetes 的编排能力扩展到原生资源（如 Pod、Deployment 等）之外的业务场景。

## CRD 的核心概念
1. 自定义资源（Custom Resource，CR）
类似 Kubernetes 原生资源（如 Pod、Service），但由用户定义，用于描述特定业务对象（如数据库实例、AI 模型、自定义工作流等）。例如，可定义一个 Database 资源，用于描述数据库的规格、存储大小等。
2. CRD（CustomResourceDefinition）
是 CR 的 “元数据定义”，用于告诉 Kubernetes API 服务器：“我要新增一种名为 <资源名> 的资源，它包含这些字段和规则”。
简单说：CRD 是 “资源的模板”，CR 是 “基于模板创建的实例”。
3. 作用
- 扩展 Kubernetes 能力，适配特定业务场景（如数据库管理、CI/CD 流程等）。
- 让自定义资源像原生资源一样通过 kubectl 命令管理，或通过 API 操作。

## CRD 的结构与示例
一个 CRD 定义包含以下核心部分：
- group：资源所属的 API 组（类似 apps、batch）。
- versions：资源的版本（如 v1、v1beta1）。
- names：资源的名称（包括单数、复数、简称等，用于 kubectl 命令）。
- schema：资源的字段定义（通过 OpenAPI v3 规范约束字段类型、默认值等）。
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com  # CRD名称，格式：<复数资源名>.<group>
spec:
  group: example.com  # 自定义API组
  names:
    kind: Database    # 资源类型（首字母大写）
    plural: databases # 复数名称（用于kubectl get databases）
    singular: database # 单数名称
    shortNames: [db]  # 简称（用于kubectl get db）
  scope: Namespaced  # 资源作用域（Namespaced/Cluster）
  versions:
  - name: v1  # 版本
    served: true  # 是否可通过API访问
    storage: true  # 是否为存储版本（同一group下仅一个）
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:  # 自定义资源的spec字段
            size:  # 数据库大小
              type: integer
              minimum: 1
              maximum: 100
            engine:  # 数据库引擎
              type: string
              enum: [mysql, postgres, mongodb]  # 允许的值
            storage:  # 存储大小
              type: string
              default: "10Gi"  # 默认值

```

## 基于 CRD 创建自定义资源（CR）
```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: mysql-db
  namespace: default
spec:
  size: 2  # 实例数量
  engine: mysql  # 数据库引擎
  storage: "20Gi"  # 存储大小
```

## 相关kubectl操作
1. 管理CRD
```bash
# 查看所有 CRD
kubectl get crd

# 查看指定 CRD（如 databases.example.com）
kubectl get crd databases.example.com
kubectl describe crd databases.example.com  # 查看详细定义

# 创建 CRD
kubectl apply -f database-crd.yaml

# 删除 CRD（会同时删除所有基于该CRD的CR实例）
kubectl delete crd databases.example.com
```

2. 管理自定义资源（CR）
```bash
# 查看指定类型的CR（如 databases）
kubectl get databases  # 或简称 kubectl get db
kubectl get databases -o yaml  # 查看详细信息

# 查看单个CR实例
kubectl get database mysql-db
kubectl describe database mysql-db

# 创建CR实例
kubectl apply -f mysql-instance.yaml

# 更新CR实例（直接编辑或替换文件）
kubectl edit database mysql-db
kubectl apply -f mysql-instance.yaml  # 重新应用修改后的文件

# 删除CR实例
kubectl delete database mysql-db
```