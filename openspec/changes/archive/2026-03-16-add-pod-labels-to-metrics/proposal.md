## Why

在 K8s 集群中，用户需要在 GPU 指标中关联 Pod 的自定义 labels（如 `app`、`team`、`env` 等），以便在 Grafana/Prometheus 中按业务维度聚合和筛选 GPU 使用情况。不需要输出所有 labels，需要支持通配符/正则匹配来过滤。

**经过代码分析，发现此功能已经在 dcgm-exporter 中实现**，只需正确配置部署参数即可启用。

## What Changes

本 change 不需要修改代码，仅需提供部署配置指南：

- **启用 Pod Labels 输出**: 通过 `--kubernetes-enable-pod-labels` 或 `DCGM_EXPORTER_KUBERNETES_ENABLE_POD_LABELS=true`
- **配置正则过滤**: 通过 `--kubernetes-pod-label-allowlist-regex` 或 `DCGM_EXPORTER_KUBERNETES_POD_LABEL_ALLOWLIST_REGEX` 环境变量设置正则模式
- **Helm Chart 部署**: 通过 `kubernetes.enablePodLabels` 和 `kubernetes.podLabelAllowlistRegex` values 配置

### 已有功能清单

| 功能 | 配置项 | 状态 |
|------|--------|------|
| Pod Labels 输出 | `kubernetes.enablePodLabels: true` | 已实现 |
| Label 正则过滤 | `kubernetes.podLabelAllowlistRegex` | 已实现 |
| LRU 缓存优化 | `KubernetesPodLabelCacheSize` | 已实现 |
| RBAC 自动创建 | `kubernetes.rbac.create: true` | 已实现 |
| Pod Informer 缓存 | 基于 NODE_NAME 节点级过滤 | 已实现 |

## Capabilities

### New Capabilities
- `deployment-guide`: 提供 Helm Chart 部署配置指南，包含 Pod labels 过滤的完整配置示例

### Modified Capabilities
（无需修改现有功能）

## Impact

- **代码**: 无需修改
- **部署**: 需要正确配置 Helm values 或环境变量
- **RBAC**: 启用 Pod Labels 需要 ClusterRole 具有 pods 的 list/watch 权限（Helm Chart 已自动处理）
- **性能**: Label 过滤使用 LRU 缓存（默认 150k 条目），对大规模集群友好
