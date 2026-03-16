# dcgm-exporter Pod Labels 部署指南

## 功能说明

dcgm-exporter 支持在 GPU 指标中输出使用 GPU 的 Pod 的 labels，并支持通过正则表达式过滤，只输出匹配的 labels。

## 快速部署

### 1. 使用 Helm Chart 部署（推荐）

```bash
# 使用预配置的 values 文件部署
helm install dcgm-exporter deployment/ \
  -f deployment/values-pod-labels.yaml \
  -n monitoring --create-namespace
```

### 2. 自定义正则过滤

编辑 `deployment/values-pod-labels.yaml` 中的 `podLabelAllowlistRegex` 列表，添加需要的正则模式：

```yaml
kubernetes:
  enablePodLabels: true
  podLabelAllowlistRegex:
    - "^app$"                         # 精确匹配 "app"
    - "^app\\.kubernetes\\.io/.*"     # app.kubernetes.io/ 前缀
    - "^(team|env|version)$"          # 精确匹配多个 label key
    - "^custom-label-.*"              # 自定义前缀匹配
```

### 3. 不使用 Helm（直接设置环境变量）

如果不使用 Helm，可以直接在 DaemonSet 中设置环境变量：

```yaml
env:
  - name: DCGM_EXPORTER_KUBERNETES
    value: "true"
  - name: DCGM_EXPORTER_KUBERNETES_ENABLE_POD_LABELS
    value: "true"
  - name: DCGM_EXPORTER_KUBERNETES_POD_LABEL_ALLOWLIST_REGEX
    value: "^app$,^app\\.kubernetes\\.io/.*,^(team|env|version)$"
  - name: NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
```

## 验证部署

### 检查 Pod 状态

```bash
kubectl get pods -n monitoring -l app.kubernetes.io/name=dcgm-exporter
```

### 查看指标输出

```bash
# 获取 dcgm-exporter Pod 名称
POD=$(kubectl get pods -n monitoring -l app.kubernetes.io/name=dcgm-exporter -o jsonpath='{.items[0].metadata.name}')

# 端口转发
kubectl port-forward -n monitoring $POD 9400:9400 &

# 查看指标（Pod labels 会作为额外的 label 出现在指标中）
curl -s http://localhost:9400/metrics | grep DCGM_FI_DEV_GPU_UTIL
```

### 预期输出示例

启用 Pod Labels 后，指标会包含匹配的 labels：

```
# 未启用 Pod Labels
DCGM_FI_DEV_GPU_UTIL{gpu="0",UUID="GPU-xxxx",pod="my-pod",namespace="default",container="main"} 85

# 启用 Pod Labels + 过滤 ^app$, ^team$
DCGM_FI_DEV_GPU_UTIL{gpu="0",UUID="GPU-xxxx",pod="my-pod",namespace="default",container="main",app="my-app",team="ml-infra"} 85
```

## 正则语法参考

dcgm-exporter 使用 Go 的 `regexp` 包（RE2 语法），常见模式：

| 需求 | 正则 | 说明 |
|------|------|------|
| 精确匹配 `app` | `^app$` | `^` 和 `$` 锚定首尾 |
| 以 `app` 开头 | `^app.*` | `.*` 匹配任意字符 |
| 以 `kubernetes.io/` 开头 | `^.*kubernetes\\.io/.*` | `.` 需要转义为 `\\.` |
| 匹配多个 key | `^(app\|team\|env)$` | `\|` 表示或 |
| 匹配所有 labels | 留空列表 `[]` | 空列表 = 不过滤 |

## RBAC 权限

启用 Pod Labels 需要 ServiceAccount 具有以下权限：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["list", "watch"]
```

Helm Chart 在 `kubernetes.rbac.create: true` 时会自动创建此 ClusterRole。

## 故障排查

1. **Labels 未出现在指标中**
   - 确认 `DCGM_EXPORTER_KUBERNETES_ENABLE_POD_LABELS=true` 已设置
   - 检查 RBAC 权限：`kubectl auth can-i list pods --as=system:serviceaccount:monitoring:dcgm-exporter`
   - 检查日志：`kubectl logs -n monitoring <pod> | grep -i label`

2. **某些 Labels 被过滤**
   - 检查 `podLabelAllowlistRegex` 正则是否正确匹配 label key
   - 注意：label key 中的 `.` 在正则中需要转义为 `\\.`

3. **性能问题**
   - Pod Labels 使用 Informer 缓存，Label 过滤使用 LRU 缓存（默认 150k 条目）
   - 如果集群规模很大，可通过 `NODE_NAME` 环境变量限制只监听本节点 Pod
