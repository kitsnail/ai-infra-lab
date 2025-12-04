# llm-d 快速入门指南

## 🚀 快速部署

### 前置要求

| 组件 | 最低要求 | 推荐配置 |
|------|----------|----------|
| **Kubernetes** | >= 1.29 | 1.30+ |
| **硬件** | NVIDIA A100 40GB | NVIDIA H100 80GB |
| **CPU** | 64核 + 64GB RAM | 128核 + 128GB RAM |

### 一键部署 (推荐)

```bash
# 步骤1: 模型准备
cd workspace/llm
kubectl apply -k .
kubectl exec -it llm-dev-xxx -n llm -- bash
modelscope download --model deepseek-ai/DeepSeek-V3.1 --local_dir /models/deepseek-ai/DeepSeek-V3.1
modelscope download --model Qwen/Qwen3-VL-235B-A22B-Instruct --local_dir /models/Qwen/Qwen3-VL-235B-A22B-Instruct

# 步骤2: 一键部署
cd workspace/llm-d-0.4.0
helmwave up --build
```

### 验证部署

```bash
# 检查Pod状态
kubectl get pods -n llm-d

# 检查服务状态
kubectl get svc -n llm-d

# 测试API
kubectl port-forward <gateway-pod> 8080:8080 -n llm-d
curl http://localhost:8080/v1/models
```

## 📋 三种部署模式

### 1. 智能推理调度（新手推荐）

```bash
# 一键部署
cd workspace/llm-d-1.3.1
helmwave up --build
```

**包含组件**: EPP + Model Service + Gateway + BBR

### 2. 预填充/解码分离（大模型优化）

```bash
# 部署Prefill服务器
helmfile apply -e prefill -n llm-d

# 部署Decode服务器
helmfile apply -e decode -n llm-d

# 部署路由侧车
helmfile apply -e sidecar -n llm-d
```

**适用场景**: 长提示，TTFT优化

### 3. 宽专家并行（MoE模型）

```bash
# 部署InferencePool
helm install wide-ep-pool \
  --set modelServers.matchLabels.app=wide-ep \
  --set provider.name=istio \
  oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool \
  -n llm-d

# 部署模型服务
helm install wide-ep-model \
  --set model.name=DeepSeek-R1 \
  --set role=wide_ep \
  oci://ghcr.io/llm-d-incubation/charts/llm-d-modelservice \
  -n llm-d
```

**适用场景**: MoE模型，最大吞吐

## 🔧 多模型支持

### 部署Body-Based Router

```bash
# 安装BBR
helm install body-based-router \
  --set provider.name=istio \
  oci://registry.k8s.io/gateway-api-inference-extension/charts/body-based-routing \
  -n llm-d
```

### 配置多模型

```bash
# 部署模型A
helm install ms-model-a \
  --set model.name="Model-A-Name" \
  --set replicas=2 \
  oci://ghcr.io/llm-d-incubation/charts/llm-d-modelservice \
  -n llm-d

# 部署模型B
helm install ms-model-b \
  --set model.name="Model-B-Name" \
  --set replicas=3 \
  oci://ghcr.io/llm-d-incubation/charts/llm-d-modelservice \
  -n llm-d
```

## 🎯 硬件适配

| 硬件类型 | 部署命令 | 说明 |
|----------|----------|------|
| **GPU** | `helmfile apply -n llm-d` | 默认配置 |
| **CPU** | `helmfile apply -e cpu -n llm-d` | CPU推理 |
| **XPU** | `helmfile apply -e xpu -n llm-d` | Intel XPU |
| **TPU** | `helmfile apply -e gke_tpu -n llm-d` | Google TPU |
| **AWS** | `helmfile apply -e aws -n llm-d` | AWS EKS |

## 📊 性能测试

### 基础测试

```bash
# 获取Gateway IP
GATEWAY_IP=$(kubectl get svc llm-d-infra-inference-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# 测试推理请求
curl -X POST http://${GATEWAY_IP}/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3-VL-235B-A22B-Instruct",
    "messages": [{"role": "user", "content": "Hello, world!"}]
  }'
```

### 基准测试

```bash
# 使用内置基准测试工具
cd llm-d-benchmark
python benchmark.py \
  --endpoint http://${GATEWAY_IP}/v1/chat/completions \
  --model "Qwen3-VL-235B-A22B-Instruct" \
  --requests 100 \
  --concurrency 10
```

## 🛠️ 故障排查

### 快速诊断

```bash
# 检查Pod状态
kubectl get pods -n llm-d
kubectl describe pod <pod-name> -n llm-d

# 检查日志
kubectl logs <epp-pod> | grep "Request handled"
kubectl logs <vllm-pod> | grep -i error

# 检查指标
kubectl port-forward <pod> 9090:9090
curl localhost:9090/metrics
```

### 常见问题

| 问题 | 解决方案 |
|------|----------|
| **Pod CrashLoopBackOff** | 检查ConfigMap配置 |
| **模型加载失败** | 检查HF_TOKEN和PVC |
| **路由失败** | 验证HTTPRoute配置 |
| **性能差** | 检查GPU利用率 |

## 📈 性能基准

| 指标 | 优秀 | 可接受 |
|------|------|----------|
| **TTFT** | <1s | <3s |
| **TPOT** | <50ms/token | <100ms/token |
| **缓存命中率** | >30% | >10% |
| **GPU利用率** | >80% | >60% |

## 🔄 版本管理

### 升级部署

```bash
# 获取最新版本
git fetch --tags
git checkout v0.4.0

# 升级部署
helmfile apply -n llm-d
```

### 回滚部署

```bash
# 回滚到稳定版本
helm rollback <release-name> <revision> -n llm-d
```

### 清理部署

```bash
# 卸载所有组件
helmfile destroy -n llm-d

# 清理命名空间
kubectl delete namespace llm-d
```

## 📚 下一步

- 参考[架构总览](../architecture/01-overview.md)了解系统架构
- 查看[组件详解](../components/)深入了解各组件
- 阅读[故障排查](../troubleshooting/01-common-issues.md)解决问题
- 探索[最佳实践](#)优化性能
