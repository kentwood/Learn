## 一、Helm 仓库管理（工程基础 / 考题高频）
### 1. 添加仓库
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```
- 解释：将第三方仓库（如 Bitnami，提供 Nginx、MySQL 等常用 Chart）添加到本地 Helm 仓库列表，后续可直接从该仓库拉取 Chart。
- 场景：工程中首次使用某类中间件（如用 Bitnami 的 Redis）；CKA 考题中 “部署指定仓库的 Chart” 前置操作。
### 2. 更新仓库缓存
```bash
helm repo update
```
- 解释：同步本地仓库缓存与远程仓库的最新元数据（如 Chart 版本更新、新增 Chart），避免拉取到旧版本。
- 场景：工程中安装 Chart 前确保版本最新；CKA 考题中 “升级 Chart” 前的必做步骤。
### 3. 查看仓库列表
```bash
helm repo list
```
- 解释：列出本地已添加的所有 Helm 仓库，包含仓库名称、URL。
- 场景：工程中确认仓库是否已添加；CKA 考题中 “验证仓库配置是否正确”。
### 4. 删除无效仓库
```bash
helm repo remove old-repo
```
- 解释：移除本地不再使用的仓库（如仓库 URL 变更、停止维护），清理本地配置。
- 场景：工程中仓库迁移；CKA 考题中 “修正错误添加的仓库”。

## 二、Chart 查询与下载（工程选型 / 考题准备）
### 1. 搜索仓库中的 Chart
```bash
helm search repo nginx
```
- 解释：在本地已添加的仓库中搜索名称包含 “nginx” 的 Chart，显示 Chart 名称、版本、描述。
- 场景：工程中筛选可用的中间件 Chart；CKA 考题中 “找到指定应用的 Chart”。
### 2. 查看 Chart 详情（含配置变量）
```bash
helm show values bitnami/nginx --version 15.3.0
```
- 解释：查看指定仓库（bitnami）、指定版本（15.3.0）的 nginx Chart 的默认配置变量（即values.yaml内容），用于后续自定义配置。
- 场景：工程中提前规划自定义参数（如修改 nginx 端口、资源限制）；CKA 考题中 “获取 Chart 的可配置参数”。
### 3. 下载 Chart 到本地（不安装）
```bash
helm pull bitnami/mysql --version 9.4.0 --untar
```
- 解释：将指定版本的 mysql Chart 下载到本地并自动解压（--untar），方便查看模板文件（templates/）或修改values.yaml。
- 场景：工程中需自定义 Chart 模板（如添加 Sidecar 容器）；CKA 考题中 “离线修改 Chart 配置后安装”。
## 三、Release 部署与运维（工程核心 / 考题重点）
### 1. 安装 Release（基础版）
```bash
helm install my-nginx bitnami/nginx --namespace nginx-namespace --create-namespace
```
- 解释：
  - my-nginx：自定义的 Release 名称（唯一标识该部署实例）；
  - bitnami/nginx：指定从 bitnami 仓库拉取 nginx Chart；
  - --namespace：指定部署到nginx-namespace命名空间；
  - --create-namespace：若命名空间不存在则自动创建。
- 场景：工程中首次部署应用；CKA 考题中 “在指定命名空间部署 Chart”。
### 2. 安装 Release（自定义配置版）
```bash
helm install my-mysql bitnami/mysql \
  --namespace mysql-namespace \
  --set auth.rootPassword=MyP@ss123 \
  --set primary.persistence.size=10Gi \
  -f custom-values.yaml
```
- 解释：
  - --set：直接在命令行自定义单个配置（如 root 密码、持久化存储大小）；
  - -f：通过本地自定义配置文件（custom-values.yaml）批量覆盖默认参数（优先级高于--set）。
- 场景：工程中个性化部署（如修改数据库密码、调整资源）；CKA 考题中 “按要求配置 Chart 参数后安装”。
### 3. 查看已安装的 Release
```bash
helm list -n mysql-namespace
```
- 解释：列出指定命名空间（mysql-namespace）下所有已安装的 Release，包含名称、Chart 版本、状态（如deployed）、更新时间。
- 场景：工程中确认应用是否部署成功；CKA 考题中 “验证 Release 是否存在”。
### 4. 升级 Release（修改配置）
```bash
helm upgrade my-nginx bitnami/nginx \
  --namespace nginx-namespace \
  --set service.type=NodePort \
  --version 15.4.0
```
- 解释：
  - 升级my-nginx Release，可同时实现 “配置修改”（如将 Service 类型从 ClusterIP 改为 NodePort）和 “Chart 版本升级”（从旧版到 15.4.0）；
  - 若仅修改配置，可不指定--version（保持当前 Chart 版本）。
- 场景：工程中调整应用配置或升级版本；CKA 考题中 “升级 Release 并修改指定参数”。
### 5. 回滚 Release（版本回退）
```bash
# 1. 查看Release的历史版本
helm history my-nginx -n nginx-namespace
# 2. 回滚到上一个版本
helm rollback my-nginx 0 -n nginx-namespace
# 或回滚到指定历史版本（如版本2）
helm rollback my-nginx 2 -n nginx-namespace
```
- 解释：
  - helm history：查看 Release 的所有部署历史（含版本号、操作时间、状态）；
  - helm rollback：将 Release 回滚到指定历史版本（0代表上一个版本），适用于升级失败后恢复。
- 场景：工程中升级出现故障（如应用启动失败）；CKA 考题中 “升级 Release 后验证失败，回滚到原版本”。
### 6. 卸载 Release
```bash
helm uninstall my-nginx -n nginx-namespace
```
- 解释：删除指定命名空间下的my-nginx Release，同时清理对应的 Kubernetes 资源（Pod、Service、Deployment 等）。
- 场景：工程中下线应用；CKA 考题中 “删除指定 Release”。
## 四、调试与验证（工程排错 / 考题验证）
### 1. 渲染模板（不部署，验证配置）
```bash
helm template my-nginx bitnami/nginx \
  --namespace nginx-namespace \
  --set service.type=NodePort \
  --dry-run
```
- 解释：根据指定配置渲染 Chart 模板，生成最终的 Kubernetes YAML 清单（如 Deployment、Service），但不实际部署（--dry-run），用于提前验证配置是否正确。
- 场景：工程中排查配置错误（如参数格式错误导致模板渲染失败）；CKA 考题中 “验证自定义配置是否符合预期”。
### 2. 查看 Release 的详细信息
```bash
helm status my-mysql -n mysql-namespace
```
- 解释：显示指定 Release 的详细状态，包含部署的资源列表（Pod 名称、Service 地址）、配置参数、健康状态等，用于确认部署是否正常。
- 场景：工程中排错（如 Pod 启动失败时查看状态）；CKA 考题中 “验证 Release 部署成功且资源正常”。
### 3. 导出 Release 的配置
```bash
helm get values my-nginx -n nginx-namespace -o yaml > exported-values.yaml
```
- 解释：导出指定 Release 当前使用的配置（即生效的values参数）到本地文件（exported-values.yaml），用于备份或复用配置。
- 场景：工程中迁移配置（如在另一环境复用相同配置）；CKA 考题中 “获取已部署 Release 的配置并保存”。
