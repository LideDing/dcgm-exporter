## 1. 创建自定义 Helm Values 配置文件

- [x] 1.1 创建 `deployment/values-pod-labels.yaml` 覆盖文件，包含以下配置：
  - `kubernetes.enablePodLabels: true` 启用 Pod Labels
  - `kubernetes.podLabelAllowlistRegex` 配置正则过滤模式示例
  - `kubernetes.rbac.create: true` 确保 RBAC 权限自动创建

## 2. 部署验证

- [x] 2.1 编写部署命令文档：`helm install` 命令示例，使用自定义 values 文件
- [x] 2.2 验证指标输出：curl metrics 端点，确认 Pod labels 出现在指标中（文档中包含验证步骤）
- [x] 2.3 验证正则过滤：确认只有匹配正则的 labels 出现在输出中（文档中包含验证步骤和故障排查）
