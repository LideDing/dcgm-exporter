## Context

dcgm-exporter 已经实现了完整的 Pod Labels 输出和正则过滤功能。核心实现位于：

- `internal/pkg/transformation/kubernetes.go` - PodMapper，通过 K8s Informer 获取 Pod metadata
- `internal/pkg/transformation/types.go` - LabelFilterCache，LRU 缓存实现
- `internal/pkg/appconfig/types.go` - 配置结构体
- `pkg/cmd/app.go` - CLI 参数定义
- `deployment/` - Helm Chart

数据流：Kubelet PodResources API → PodMapper.getMappings() → createPodInfo() (通过 Informer 获取 Pod Labels) → shouldIncludeLabel() (正则过滤 + LRU 缓存) → Process() (将 Labels 注入 Metrics)

## Goals / Non-Goals

**Goals:**
- 提供完整的 Helm Chart 部署配置，启用 Pod Labels 输出和正则过滤
- 提供常见 label 过滤模式的配置示例
- 确保 RBAC 权限正确配置

**Non-Goals:**
- 不需要修改任何 Go 代码
- 不需要添加新的命令行参数（已有）
- 不涉及非 K8s 环境的部署

## Decisions

1. **使用 Helm Chart 部署**：Helm Chart 已经完整支持所有配置项，通过 values.yaml 覆盖即可
2. **正则匹配而非通配符**：dcgm-exporter 使用 Go 正则表达式（`regexp` 包），不是 glob 通配符。用户需求中的"通配符"需转换为正则表达式：
   - `app*` → `^app.*`
   - `team` (精确匹配) → `^team$`
   - `app.kubernetes.io/*` → `^app\.kubernetes\.io/.*`
3. **环境变量传递正则**：多个正则模式通过逗号分隔传递给 `DCGM_EXPORTER_KUBERNETES_POD_LABEL_ALLOWLIST_REGEX`

## Risks / Trade-offs

- **性能**: 启用 Pod Labels 会增加 Kubernetes API 调用和内存使用。Informer 缓存和 LRU label 过滤缓存已做优化
- **RBAC 权限**: 需要 ClusterRole 具有 pods 的 list/watch 权限，Helm Chart 会自动创建
- **Label 基数**: 如果正则匹配的 labels 值变化频繁（如包含时间戳），可能导致 Prometheus 时间序列爆炸。建议只匹配稳定的业务 labels
