# llm-d 常见故障排查指南

## 🚨 快速诊断

```bash
# 基础状态检查
kubectl get pods -n llm-d
kubectl get svc -n llm-d
kubectl get events -n llm-d --sort-by=.metadata.creationTimestamp | tail -10

# 关键日志检查
kubectl logs -f <epp-pod> | grep "Request handled"
kubectl logs -f <vllm-pod> | grep -i error
kubectl logs -f body-based-router-xxx | grep -E "(Processed|error)"
```

## 📋 部署类问题

### Pod启动失败

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **ImagePullBackOff** | 镜像拉取失败 | 检查网络、镜像仓库权限 |
| **CrashLoopBackOff** | 配置错误或资源不足 | 检查ConfigMap、资源限制 |
| **Pending** | 调度失败 | 检查资源配额、污点容忍度 |

### helmwave部署问题 (v1.3.1)

```bash
# 检查helmwave状态
helmwave status

# 重新构建部署
helmwave up --build --force

# 检查依赖
helmwave deps
```

### PVC存储问题

```bash
# 检查PVC状态
kubectl get pvc -n llm-d
kubectl describe pvc llm-model -n llm-d

# 检查模型下载
kubectl exec -it <model-pod> -- ls -la /model-cache
```

## 🌐 网络类问题

### 服务发现失败

```bash
# 测试内部连接
kubectl exec -it <pod-name> -- nslookup <service-name>.llm-d.svc.cluster.local

# 检查Endpoints
kubectl get endpoints -n llm-d
```

### Gateway路由问题

```bash
# 检查Gateway配置
kubectl get gateway -n llm-d
kubectl get httproute -n llm-d -o yaml

# 检查Envoy配置
kubectl exec -it <istio-gateway> -- curl localhost:15000/config_dump
```

### ExtProc链问题

| 问题 | 检查方法 | 解决方案 |
|------|----------|----------|
| **BBR无响应** | `kubectl logs body-based-router | grep error` | 检查Service discovery |
| **EPP路由失败** | `kubectl logs gaie-xxx-epp | grep "Request handled"` | 验证插件配置 |
| **链式顺序错误** | 检查EnvoyFilter patch顺序 | 确保BBR在前，EPP在后 |

## 🤖 模型推理问题

### 模型加载失败

```bash
# 检查加载日志
kubectl logs <vllm-pod> | grep -E "(Loading|Error|Failed)"

# 检查HF Token
kubectl get secret llm-d-hf-token -o yaml

# 检查GPU状态
kubectl exec -it <vllm-pod> -- nvidia-smi
```

### 性能问题

| 指标 | 问题诊断 | 解决方案 |
|------|----------|----------|
| **TTFT > 5s** | 模型预加载或缓存问题 | 检查预热配置，启用PD分离 |
| **TPOT > 200ms** | 批处理或GPU利用率低 | 优化批处理大小，检查GPU使用 |
| **缓存命中率 < 10%** | prefix-cache-scorer未启用 | 切换到V2插件 |

## 💾 缓存系统问题

### KV缓存问题

```bash
# 检查缓存状态
kubectl exec -it <kv-cache-pod> -- curl localhost:8080/cache/status

# 检查缓存指标
kubectl port-forward <kv-cache-pod> 9090:9090
curl http://localhost:9090/metrics | grep cache_hit_ratio
```

### 缓存优化检查

| 问题 | 优化建议 |
|------|----------|
| **命中率低** | 调整block_size和max_prefix_blocks_to_match |
| **内存过高** | 减少cache容量或调整gc_threshold |
| **同步失败** | 检查网络连通性和权限配置 |

## 🛠️ 故障排查工具

### 诊断脚本

```bash
#!/bin/bash
# llm-d-diagnostics.sh
NAMESPACE=${1:-llm-d}

echo "=== llm-d 诊断报告 ==="
echo "时间: $(date)"
echo ""

# Pod状态
echo "=== Pod状态 ==="
kubectl get pods -n $NAMESPACE -o wide

# 服务状态
echo "=== 服务状态 ==="
kubectl get svc -n $NAMESPACE

# 资源使用
echo "=== 资源使用 ==="
kubectl top pods -n $NAMESPACE --sort-by=cpu

# 最近事件
echo "=== 最近事件 ==="
kubectl get events -n $NAMESPACE --sort-by=.metadata.creationTimestamp | tail-10
```

### 性能监控

```bash
# 安装Prometheus监控
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install grafana prometheus-community/grafana -n monitoring

# 访问指标
kubectl port-forward <pod> 9090:9090
curl http://localhost:9090/metrics
```

## 🚨 应急处理

### 快速恢复

```bash
# 重启所有组件
kubectl rollout restart deployment -n llm-d

# 强制重新调度
kubectl delete pods --field-selector=status.phase!=Running -n llm-d

# 清理缓存（如缓存损坏）
kubectl exec -it <kv-cache-pod> -- rm -rf /cache/*
```

### 服务降级

```yaml
fallback:
  enabled: true
  conditions:
    - error_rate > 0.1
    - response_time_p99 > 30s
  actions:
    - switch_to_simple_routing
    - disable_caching
    - reduce_batch_size
```

### 容量扩展

```bash
# 自动扩容
kubectl autoscale deployment gaie-xxx-epp \
  --cpu-percent=70 \
  --min=2 \
  --max=10 \
  -n llm-d

# 手动扩容
kubectl scale deployment ms-xxx-decode --replicas=5 -n llm-d
```

## 📞 获取帮助

### 信息收集

```bash
# 收集诊断信息
kubectl get all -n llm-d > cluster-state.txt
kubectl describe pods -n llm-d >> cluster-state.txt
kubectl logs --all-containers=true -n llm-d >> logs.txt
kubectl get events -n llm-d >> events.txt

# 打包发送
tar -czf llm-d-diagnostics.tar.gz cluster-state.txt logs.txt events.txt
```

### 支持渠道

- **GitHub Issues**: https://github.com/llm-d/llm-d/issues
- **Slack**: #llm-d-community
- **企业支持**: support@llm-d.io (2小时响应)

## 🔄 预防性维护

### 健康检查

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: llm-d-health-check
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: health-check
            image: curlimages/curl
            command:
            - /bin/sh
            - -c
            - |
              curl -f http://gateway:8080/health || exit 1
```

### 告警配置

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: llm-d-alerts
spec:
  groups:
  - name: llm-d
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
      for: 2m
    - alert: LowCacheHitRatio
      expr: cache_hit_ratio < 0.1
      for: 5m
