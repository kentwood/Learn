## 为什么需要三向合并
在 Helm 2 中，升级 Release 时采用的是 “简单替换” 逻辑：直接用新生成的 YAML 配置覆盖集群中已有的资源。这会导致一个问题：如果用户手动修改过资源（比如临时调整副本数、修改环境变量），这些手动变更会被 Helm 升级完全覆盖，引发意外。

Helm 3 引入 “三向合并” 后，通过对比三个关键版本的配置，智能合并变更，既保留手动修改，又应用新的 Chart 配置，避免了 “一刀切” 的覆盖。

## 三向合并的三角
1. 基础版本（Base）
   - 指上一次通过 Helm 部署 / 升级时，生成并提交到集群的资源配置（即上一版本 Release 的 manifest）。
   - 存储在 Helm 的 Release 历史记录中（可通过 helm history <release-name> 查看）。
2. 本地新版本（Local）
   - 指本次执行 helm upgrade 时，根据当前 Chart 模板和 Values 配置生成的新资源配置（即将部署的目标配置）。
3. 集群当前版本（Remote）
   - 指集群中当前实际运行的资源配置（可能包含用户通过 kubectl edit 等方式做的手动修改）。

## 冲突合并逻辑
- 若 Base 和 Remote 相同（无手动修改）：直接使用 Local 版本（新配置）。
- 若 Base 和 Remote 不同（有手动修改）：
  - 若 Local 与 Base 相同（本次升级未修改该字段）：保留 Remote 的手动修改。
  - 若 Local 与 Base 不同（本次升级修改了该字段）：
    - 若 Remote 与 Base 不同（手动也改了该字段）：触发冲突，Helm 会优先使用 Local 版本（新配置），并在日志中提示冲突。
    - 若 Remote 与 Base 相同（手动未改该字段）：使用 Local 版本（新配置）。

## 升级与冲突示例
### 1. 初始部署（生成 Base 版本）
```text
# 部署 Release，初始副本数 1（来自 values.yaml）
helm install my-nginx bitnami/nginx
```
此时 Base 版本的 Deployment 副本数为 1。
### 2. 手动修改资源（生成 Remote 版本）
```text
# 升级 Release，修改镜像版本（不碰副本数）
helm upgrade my-nginx bitnami/nginx \
  --set image.tag=1.25.3  # 仅修改镜像版本
```
此时集群中 Remote 版本的副本数为 3（Base 是 1，Remote 被手动修改）。
### 3. Helm 升级（生成 Local 版本）
```text
# 升级 Release，修改镜像版本（不碰副本数）
helm upgrade my-nginx bitnami/nginx \
  --set image.tag=1.25.3  # 仅修改镜像版本
```
此时 Local 版本的副本数仍为 1（未修改），但镜像版本更新为 1.25.3。
### 4. 三向合并结果
Helm 对比三个版本：

- Base 副本数 = 1，Local 副本数 = 1（未变），Remote 副本数 = 3（手动修改）→ 保留 Remote 的 3。
- 镜像版本：Base = 旧版本，Local=1.25.3，Remote = 旧版本（未手动改）→ 应用 Local 的 1.25.3。

最终部署的 Deployment：副本数 = 3（保留手动修改），镜像 = 1.25.3（应用新配置），实现了两者的融合。