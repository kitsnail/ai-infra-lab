# Model Service 组件详解

## 核心功能

Model Service是llm-d的模型服务管理组件，通过Helm Chart提供声明式Kubernetes资源管理，支持模块化、可重现、可扩展的LLM部署。

**关键特性**: 与vLLM/SGLang深度集成，支持多硬件架构

## 角色管理

| 角色 | 功能 | 适用场景 |
|------|------|----------|
| **prefill** | 提示处理 + KV缓存生成 | 大模型长提示优化 |
| **decode** | 令牌生成 + 响应完成 | 高吞吐解码服务 |
| **both** | 完整推理流水线 | 单体模式部署 |

## 硬件支持

| 硬件 | 镜像标签 | 推荐配置 |
|------|----------|----------|
| **NVIDIA GPU** | `:v0.4.0-cuda` | H100 80GB + NVLink |
| **Intel XPU** | `:v0.4.0-xpu` | Data Center GPU Max |
| **Google TPU** | `:v0.4.0-gke` | TPU v5e/v5p |
| **AWS EKS** | `:v0.4.0-aws` | EFA优化 |
| **CPU** | `:v0.4.0-cpu` | 64核 + 64GB RAM |

## 实战配置 (v1.3.1验证)

### DeepSeek V3.1 配置
```yaml
# SGLang + TP8 + FP8 量化
modelArtifacts:
  uri: "pvc://llm-model/deepseek-ai/DeepSeek-V3.1"
  name: DeepSeek-V3.1

decode:
  create: true
  replicas: 1
  containers:
    - name: sglang-cr
      image: 100.125.1.248/lmsysorg/sglang:v0.5.2-cu129-b200
      args:
        - >
          python3 -m sglang.launch_server 
          --host 0.0.0.0 
          --port 8200 
          --model-path /model-cache/deepseek-ai/DeepSeek-V3.1 
          --served-model-name DeepSeek-V3.1 
          --tp 8 
          --context-length 131072 
          --mem-fraction-static 0.82
          --chunked-prefill-size 4096
          --quantization fp8
          --tool-call-parser deepseekv31
      resources:
        requests:
          nvidia.com/gpu: 8
          memory: 2Ti
```

### Qwen3-VL-235B 配置
```yaml
# vLLM + 2Pod4G + KV缓存优化
modelArtifacts:
  uri: "pvc://llm-model/Qwen/Qwen3-VL-235B-A22B-Instruct"
  name: Qwen3-VL-235B-A22B-Instruct

decode:
  create: true
  replicas: 2
  containers:
    - name: vllm
      image: "vllm/vllm-openai:v0.11.1"
      args:
        - >
          python3 -m vllm.entrypoints.openai.api_server
          --model /model-cache/Qwen/Qwen3-VL-235B-A22B-Instruct
          --served-model-name Qwen3-VL-235B-A22B-Instruct
          --tensor-parallel-size 4
          --max-model-len 262144
          --enable-chunked-prefill
          --gpu-memory-utilization 0.85
          --kv-cache-datatype "fp8"
          --enable-prefix-caching
      resources:
        requests:
          nvidia.com/gpu: "4"
          memory: 1Ti
```

## 部署方式

### Helmwave部署
```bash
# 一键部署
cd workspace/llm-d-1.3.1
helmwave up --build

# 包含Model Service和InferencePool
repositories:
  - name: llm-d-modelservice
    url: https://llm-d-incubation.github.io/llm-d-modelservice/

releases:
  - name: ms-ds-v3-1
    chart: llm-d-modelservice/llm-d-modelservice
    version: v0.2.7
    values:
      - ms-ds-v3-1/values-sg-v3.yaml
    namespace: llm
```

### InferenceModel配置
```yaml
apiVersion: inference.networking.x-k8s.io/v1alpha1
kind: InferenceModel
metadata:
  name: qwen3-235b-model
spec:
  modelName: "Qwen3-VL-235B-A22B-Instruct"
  modelService:
    name: ms-qwen3-235b-llm-d-modelservice
    namespace: llm
  scaling:
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilization: 70
```

## 资源配置

### GPU资源
```yaml
resources:
  requests:
    nvidia.com/gpu: 8
    memory: "128Gi"
    cpu: "16"
  limits:
    nvidia.com/gpu: 8
    memory: "256Gi"
    cpu: "32"
```

### CPU资源
```yaml
resources:
  requests:
    cpu: "64"
    memory: "64Gi"
  limits:
    cpu: "128"
    memory: "128Gi"
```

### 存储配置
```yaml
# 模型存储
storage:
  type: persistent_volume
  size: "500Gi"
  access_mode: "ReadWriteMany"
  storage_class: "ssd"

# KV缓存存储
kv_cache:
  type: tiered
  local_memory: "80%"
  local_disk: "20%"
  shared_storage: "optional"
```

## 性能优化

### 推理参数
```yaml
# 批处理配置
batching:
  max_batch_size: 32
  max_num_batched_tokens: 8192
  max_num_seqs: 256

# 采样配置
sampling:
  top_k: 40
  top_p: 0.9
  temperature: 0.7
  repetition_penalty: 1.1

# KV缓存优化
kv_cache:
  enable_prefix_caching: true
  gpu_memory_utilization: 0.85
  kv_cache_datatype: "fp8"
```

## 故障排查

### 常见问题
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **Pod启动失败** | HuggingFace Token无效 | 检查HF_TOKEN Secret |
| **OOM错误** | GPU内存不足 | 调整gpu-memory-utilization或减少batch_size |
| **模型加载失败** | 网络或存储问题 | 检查PVC挂载和模型路径 |

### 调试命令
```bash
# 检查启动日志
kubectl logs -f <model-pod> | grep -E "(Loading|Error|Failed)"

# 检查GPU状态
kubectl exec -it <vllm-pod> -- nvidia-smi

# 检查指标
kubectl port-forward <vllm-pod> 8000:8000
curl http://localhost:8000/metrics | grep vllm
```

## 版本信息

- **当前版本**: v0.3.8
- **核心特性**:
  - 多硬件镜像变体
  - 声明式配置管理
  - 预加载优化
  - KV缓存集成

## 最佳实践

### 生产部署
1. **资源规划**: 根据模型大小合理分配GPU/CPU
2. **存储优化**: 使用高速SSD存储模型权重
3. **网络配置**: 确保足够带宽用于KV缓存传输
4. **监控告警**: 设置关键指标阈值告警

### 性能调优
1. **批处理大小**: 根据硬件调整批处理参数
2. **内存管理**: 优化GPU内存分配策略
3. **缓存策略**: 合理配置KV缓存层级
4. **并行策略**: 选择合适的张量/流水线并行度
