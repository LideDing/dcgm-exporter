## ADDED Requirements

### Requirement: Helm Chart 部署配置包含 Pod Labels 过滤

#### Scenario: 用户通过 Helm 部署并启用 Pod Labels 正则过滤
- **WHEN** 用户在 values.yaml 中设置 `kubernetes.enablePodLabels: true` 和 `kubernetes.podLabelAllowlistRegex` 正则列表
- **THEN** exporter 输出的 GPU 指标中包含匹配正则的 Pod labels 作为 Prometheus labels

#### Scenario: 用户使用通配符风格的过滤需求
- **WHEN** 用户想匹配以 `app` 开头的所有 labels
- **THEN** 需要配置正则 `^app.*` 而非 glob 通配符 `app*`

#### Scenario: RBAC 权限自动创建
- **WHEN** `kubernetes.enablePodLabels: true` 且 `kubernetes.rbac.create: true`
- **THEN** Helm Chart 自动创建 ClusterRole 和 ClusterRoleBinding，授予 pods 的 list/watch 权限
