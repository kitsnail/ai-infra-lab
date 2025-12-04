# llm-d 架构指南

> **llm-d v0.4.0** - 分布式AI推理服务栈架构与实战指南

## � 快速开始

### 核心特性
- **智能推理调度**: EPP + vLLM + Gateway，P90延迟改善3倍
- **多模型统一**: Body-Based Routing动态路由，无需路径分离
- **高级缓存**: KV缓存感知调度，分层存储管理
- **一键部署**: Helmwave统一部署，支持GPU/CPU/TPU/XPU

### 快速部署
```bash
# 步骤1: 模型准备
cd workspace/llm
kubectl apply -k .
kubectl exec -it llm-dev-xxx -n llm -- bash
modelscope download --model deepseek-ai/DeepSeek-V3.1 --local_dir /models/deepseek-ai/DeepSeek-V3.1
modelscope download --model Qwen/Qwen3-VL-235B-A22B-Instruct --local_dir /models/Qwen/Qwen3-VL-235B-A22B-Instruct

# 步骤2: 一键部署（推荐）
cd workspace/llm-d-0.4.0
helmwave up --build
```

## 📋 部署选择

| 场景 | 模式 | 命令 | 优势 |
|------|------|------|------|
| **新手入门** | 智能推理调度 | `helmwave up --build` | 简单快速，默认优化 |
| **生产服务** | 智能调度 + BBR | `helmwave up --build` + BBR配置 | 多模型支持，高可用 |
| **大模型** | 预填充/解码分离 | `helmfile apply -e prefill && helmfile apply -e decode` | 长提示优化，TTFT提升 |
| **MoE模型** | 宽专家并行 | `helm install wide-ep-pool ...` | 最大吞吐，专家并行 |

## 🏗️ 核心组件

| 组件 | 功能 | 关键特性 | 版本 |
|------|------|----------|------|
| **Inference Scheduler** | 智能路由决策 | 插件化调度，缓存感知 | v0.3.2 |
| **Model Service** | 模型服务管理 | 声明式部署，多硬件支持 | v0.3.8 |
| **Body-Based Router** | 多模型动态路由 | 请求体解析，统一路径 | v1.0.0 |
| **KV Cache Manager** | 缓存管理 | 分层存储，跨节点协调 | v0.3.0 |

## 🔧 实战配置

### 智能推理调度配置
```yaml
# EPP插件配置
plugins:
  - type: low-queue-filter
  - type: kv-cache-scorer
  - type: prefix-cache-scorer
    parameters:
      hashBlockSize: 64
      maxPrefixBlocksToMatch: 256
```

### 模型服务配置
```yaml
# DeepSeek V3.1 (SGLang + TP8 + FP8)
modelArtifacts:
  uri: "pvc://llm-model/deepseek-ai/DeepSeek-V3.1"
  name: DeepSeek-V3.1
resources:
  requests:
    nvidia.com/gpu: 8
    memory: 2Ti

# Qwen3-VL-235B (vLLM + 2Pod4G + KV缓存)
modelArtifacts:
  uri: "pvc://llm-model/Qwen/Qwen3-VL-235B-A22B-Instruct"
  name: Qwen3-VL-235B-A22B-Instruct
resources:
  requests:
    nvidia.com/gpu: 4
    memory: 1Ti
```

### 多模型路由配置
```yaml
# BBR + EPP 链式配置
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
spec:
  configPatches:
    - applyTo: HTTP_FILTER
      patch:
        operation: INSERT_FIRST  # BBR第一
        value:
          name: envoy.filters.http.ext_proc
          processing_mode:
            request_body_mode: FULL_DUPLEX_STREAMED
    - applyTo: HTTP_FILTER
      patch:
        operation: INSERT_AFTER   # EPP第二
        value:
          name: envoy.filters.http.ext_proc
          processing_mode:
            request_body_mode: BUFFERED

# HTTPRoute匹配
matches:
  - headers:
    - type: Exact
      name: X-Gateway-Model-Name
      value: "Model-Name"
```

## 📊 性能基准

| 指标 | 优秀 | 可接受 | 需优化 |
|------|------|----------|----------|
| **TTFT** | <1s | <3s | >5s |
| **TPOT** | <50ms/token | <100ms/token | >200ms/token |
| **缓存命中率** | >30% | >10% | <10% |
| **GPU利用率** | >80% | >60% | <40% |

## 🛠️ 故障排查

### 快速诊断
```bash
# 检查部署状态
kubectl get pods -n llm-d
kubectl get svc -n llm-d

# 检查关键日志
kubectl logs -f <epp-pod> | grep "Request handled"
kubectl logs -f <vllm-pod> | grep -i error

# 检查性能指标
kubectl port-forward <pod> 9090:9090
curl localhost:9090/metrics | grep endpoint_picker
```

### 常见问题
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **Pod CrashLoopBackOff** | ConfigMap配置错误 | 检查插件配置语法 |
| **路由失败** | HTTPRoute匹配规则错误 | 验证headers匹配配置 |
| **缓存命中率低** | prefix-cache-scorer未启用 | 切换到V2插件 |
| **OOM错误** | GPU内存不足 | 调整gpu-memory-utilization或减少batch_size |

## 📚 文档结构

### 🏗️ 架构设计
- **[01-overview.md](architecture/01-overview.md)** - 整体架构总览
- **[02-dataflow.md](architecture/02-dataflow.md)** - 数据流架构和关键路径

### 🔧 核心组件
- **[01-inference-scheduler.md](components/01-inference-scheduler.md)** - Inference Scheduler详解
- **[02-model-service.md](components/02-model-service.md)** - Model Service详解
- **[03-body-based-routing.md](components/03-body-based-routing.md)** - Body-Based Routing详解
- **[04-kv-cache-manager.md](components/04-kv-cache-manager.md)** - KV Cache Manager详解
- **[05-inference-sim.md](components/05-inference-sim.md)** - Inference Simulator详解
- **[06-benchmark-tools.md](components/06-benchmark-tools.md)** - Benchmark Tools详解

### 🚀 部署配置
- **[01-quick-start.md](deployment/01-quick-start.md)** - 快速入门指南

### 🔍 故障排查
- **[01-common-issues.md](troubleshooting/01-common-issues.md)** - 常见故障排查指南

## 📞 获取帮助

### 社区支持
- GitHub Issues: https://github.com/llm-d/llm-d/issues
- Slack: #llm-d-community
- 文档: https://llm-d.io/docs

### 企业支持
- 技术支持: support@llm-d.io
- SLA: 2小时响应，24小时解决

---

**注意**: 本指南基于llm-d v0.4.0版本编写，基于实际验证的v1.3.1部署经验。建议配合官方文档使用。
