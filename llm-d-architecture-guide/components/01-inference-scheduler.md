# Inference Scheduler (EPP) 组件详解

## 核心功能

Inference Scheduler (EPP) 是llm-d的智能路由决策组件，作为Endpoint Picker运行在Kubernetes Gateway中，通过插件化架构实现负载均衡、延迟优化和缓存感知路由。

**关键性能**: P90延迟改善可达3倍

## 插件系统

### 过滤器 (Filters)
```yaml
- type: low-queue-filter
  parameters:
    threshold: 128

- type: lora-affinity-filter
  parameters:
    threshold: 0.999

- type: least-queue-filter

- type: least-kv-cache-filter
```

### 评分器 (Scorers) - 推荐
```yaml
# V2插件 (推荐用于生产环境)
- type: queue-scorer

- type: kv-cache-scorer

- type: prefix-cache-scorer
  parameters:
    hashBlockSize: 64
    maxPrefixBlocksToMatch: 256
    lruCapacityPerServer: 31250
    cacheHitBonus: 100
```

### 决策树
```yaml
# 低延迟过滤器链
- type: decision-tree-filter
  name: low-latency-filter
  parameters:
    current: low-queue-filter
    nextOnSuccess:
      - lora-affinity-filter
      - least-queue-filter
      - least-kv-cache-filter
    nextOnFailure:
      - least-queue-filter
      - lora-affinity-filter
      - least-kv-cache-filter
```

## 部署配置

### Helmwave部署 (v1.3.1验证)
```bash
# 一键部署包含EPP
cd workspace/llm-d-1.3.1
helmwave up --build

# 或单独部署InferencePool
helm install gaie-<name> \
  oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool \
  --set provider.name=istio \
  -n llm
```

### ConfigMap配置
```yaml
apiVersion: inference.networking.x-k8s.io/v1alpha1
kind: EndpointPickerConfig
plugins: [...]
schedulingProfiles:
  - name: default
    plugins:
      - pluginRef: low-latency-filter
      - pluginRef: random-picker
```

### 监控指标
```yaml
# 关键指标
saturationDetector:
  queueDepthThreshold: 5
  kvCacheUtilThreshold: 0.8
  metricsStalenessThreshold: 200ms

metrics:
  - endpoint_picker_requests_total
  - endpoint_picker_duration_seconds
  - cache_hit_ratio
  - load_balance_score
```

## 硬件支持

| 硬件类型 | 部署命令 | 说明 |
|----------|----------|------|
| **GPU** | `helmfile apply -n ${NAMESPACE}` | 默认配置 |
| **CPU** | `helmfile apply -e cpu -n ${NAMESPACE}` | CPU推理 |
| **XPU** | `helmfile apply -e xpu -n ${NAMESPACE}` | Intel XPU |
| **TPU** | `helmfile apply -e gke_tpu -n ${NAMESPACE}` | Google TPU |

## 性能对比

| 特性 | V1 (过滤器链) | V2 (评分器) |
|------|---------------|-------------|
| **复杂度** | 简单直接 | 更灵活 |
| **精度** | 基础负载均衡 | 精确评分 |
| **缓存感知** | 有限 | 强大前缀缓存 |
| **推荐场景** | 小规模部署 | 大规模生产 |

## 故障排查

### 常见问题
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **Pod CrashLoopBackOff** | ConfigMap配置错误 | 检查插件配置语法 |
| **请求路由失败** | HTTPRoute匹配规则错误 | 验证headers匹配配置 |
| **缓存命中率低** | prefix-cache-scorer未启用 | 切换到V2插件 |

### 调试命令
```bash
# 检查EPP日志
kubectl logs -f <epp-pod> | grep "Request handled"

# 检查指标
kubectl port-forward <epp-pod> 9090:9090
curl localhost:9090/metrics

# 验证插件配置
kubectl exec -it <epp-pod> -- cat /config/default-plugins.yaml
```

## 版本信息

- **当前版本**: v0.3.2
- **核心特性**: 
  - KV缓存感知路由
  - 插件化调度架构
  - 多硬件支持
  - 与BBR链式配置

## 最佳实践

### 生产部署
1. **高可用**: 至少2个EPP副本
2. **监控**: 集成Prometheus/Grafana
3. **日志**: 启用结构化日志
4. **安全**: 启用mTLS和RBAC

### 性能调优
1. **队列阈值**: 根据硬件性能调整
2. **缓存权重**: 根据模型大小调优
3. **评分权重**: 动态调整优先级
4. **资源分配**: CPU 2核，内存4GB起
