# Inference Simulator 组件详解

## 组件概述

Inference Simulator是llm-d的轻量级vLLM模拟器，专为开发测试和性能基准测试设计。它不执行真实的模型推理，但模拟vLLM的HTTP REST API响应，支持CPU-only工作负载和多种测试场景。

## 核心功能

### 1. OpenAI兼容API模拟
- **Chat Completions**: `/v1/chat/completions` API端点
- **Completions**: `/v1/completions` API端点  
- **Models**: `/v1/models` 模型列表端点
- **LoRA管理**: `/v1/load_lora_adapter`, `/v1/unload_lora_adapter` 动态适配器管理
- **健康检查**: `/health`, `/ready` 标准健康检查端点
- **指标暴露**: `/metrics` Prometheus指标端点

### 2. 操作模式
- **Echo模式**: 返回请求中的相同文本内容
- **Random模式**: 从预定义句子集合中随机选择响应

### 3. 性能模拟
- **时间控制**: 精确的TTFT和TPOT延迟模拟
- **负载感知**: 基于并发请求的动态时间因子
- **KV缓存模拟**: 支持缓存事件和状态管理
- **故障注入**: 可配置的失败概率和类型

## 技术架构

### API端点支持
```yaml
# OpenAI兼容端点
openai_endpoints:
  - /v1/chat/completions     # 聊天完成API
  - /v1/completions        # 文本完成API
  - /v1/models            # 模型列表API

# vLLM特定端点
vllm_endpoints:
  - /v1/load_lora_adapter    # LoRA适配器加载
  - /v1/unload_lora_adapter  # LoRA适配器卸载
  - /metrics               # Prometheus指标
  - /health                # 健康检查
  - /ready                 # 就绪检查
```

### Prometheus指标支持
```yaml
# 核心推理指标
metrics:
  - vllm:gpu_cache_usage_perc     # GPU缓存使用率
  - vllm:lora_requests_info       # LoRA请求统计
  - vllm:num_requests_running     # 运行中请求数
  - vllm:num_requests_waiting     # 等待中队列请求数
  - vllm:e2e_request_latency_seconds    # 端到端延迟
  - vllm:request_inference_time_seconds   # 推理时间
  - vllm:request_queue_time_seconds      # 队列等待时间
  - vllm:request_prefill_time_seconds    # 预填充时间
  - vllm:request_decode_time_seconds     # 解码时间
  - vllm:time_to_first_token_seconds   # 首token时间
  - vllm:time_per_output_token_seconds  # 输出token时间
  - vllm:request_generation_tokens      # 生成token数
  - vllm:request_params_max_tokens     # 最大token参数
  - vllm:request_prompt_tokens         # 提示token数
  - vllm:request_success_total         # 成功请求数
```

## 配置管理

### 基础配置参数
```bash
# 必需参数
--model "model-name"              # 当前加载的模型
--port 8000                     # 监听端口

# 可选参数
--served-model-name "model1 model2"  # 暴露的模型名称列表
--mode "echo|random"              # 操作模式 (默认: random)
--max-num-seqs 5                 # 最大并发序列数
--max-waiting-queue-length 1000   # 最大等待队列长度
--max-model-len 1024             # 模型上下文窗口
```

### 性时延控制
```bash
# 时间延迟参数
--time-to-first-token 1000           # 首token延迟(ms)
--time-to-first-token-std-dev 100    # 首token延迟标准差(ms)
--inter-token-latency 50              # token间延迟(ms)  
--inter-token-latency-std-dev 10     # token间延迟标准差(ms)
--kv-cache-transfer-latency 200      # KV缓存传输延迟(ms)
--kv-cache-transfer-latency-std-dev 20 # KV缓存传输延迟标准差(ms)
--time-factor-under-load 1.0         # 负载下时间因子
```

### 预填充计算模式
```bash
# 预填充参数
--prefill-overhead 100              # 预填充固定开销(ms)
--prefill-time-per-token 2          # 每个token预填充时间(ms)
--prefill-time-std-dev 0.5         # 预填充时间标准差(ms)
--kv-cache-transfer-time-per-token 1.5 # KV缓存每token传输时间(ms)
--kv-cache-transfer-time-std-dev 0.3  # KV缓存传输时间标准差(ms)
```

### LoRA适配器配置
```bash
# LoRA参数
--lora-modules '[{"name":"adapter1","path":"/path/to/lora1","base_model_name":"base-model"}]'  # LoRA适配器列表
--max-loras 10                     # 单批次最大LoRA数
--max-cpu-loras 20                 # CPU内存中最大LoRA数
```

### KV缓存配置
```bash
# KV缓存参数
--enable-kvcache                   # 启用KV缓存支持
--kv-cache-size 100000             # 最大token块数
--block-size 64                    # token块大小 (8,16,32,64,128)
--tokenizers-cache-dir "/tmp/cache" # tokenizer缓存目录
--zmq-endpoint "tcp://localhost:5555"  # ZMQ事件发布地址
--event-batch-size 16              # 事件批处理大小
```

### 故障注入配置
```bash
# 故障注入参数
--failure-injection-rate 5          # 故障注入概率(0-100)
--failure-types "rate_limit,invalid_api_key,context_length,server_error"  # 故障类型列表
```

### 数据集集成
```bash
# 数据集参数
--dataset-path "/path/to/dataset.sqlite3"    # SQLite数据集文件路径
--dataset-url "https://example.com/dataset.sqlite3"  # 数据集下载URL
--dataset-in-memory                       # 数据集加载到内存
```

## 部署集成

### Docker镜像部署
```bash
# 构建镜像
make image-build

# 运行容器 (推荐使用最新版本)
docker run --rm --publish 8000:8000 \
  ghcr.io/llm-d/llm-d-inference-sim:v0.6.1 \
  --port 8000 \
  --model "Qwen/Qwen2.5-1.5B-Instruct" \
  --lora-modules '[{"name":"tweet-summary-0"}]' \
  --time-to-first-token 1000 \
  --inter-token-latency 50
```

### Kubernetes部署
```yaml
# 部署清单示例
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-sim
spec:
  replicas: 1
  selector:
    matchLabels:
      app: inference-sim
  template:
    metadata:
      labels:
        app: inference-sim
    spec:
      containers:
      - name: inference-sim
        image: ghcr.io/llm-d/llm-d-inference-sim:v0.6.1
        ports:
        - containerPort: 8000
        args:
        - --model "meta-llama/Llama-3.1-8B-Instruct"
        - --port 8000
        - --mode echo
        - --time-to-first-token 500
        - --inter-token-latency 30
        resources:
          requests:
            cpu: "1"
            memory: "2Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: inference-sim
spec:
  selector:
    app: inference-sim
  ports:
  - port: 8000
    targetPort: 8000
  type: ClusterIP
```

### 数据并行部署
```bash
# 多实例数据并行部署
./bin/llm-d-inference-sim \
  --model my_model \
  --port 8000 \
  --data-parallel-size 4
  # 自动分配端口: 8000, 8001, 8002, 8003
```

## 使用场景

### 1. 开发测试
```bash
# 本地测试环境启动
./bin/llm-d-inference-sim \
  --model "test-model" \
  --mode echo \
  --port 8000

# 测试API调用
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "test-model", 
    "messages": [{"role": "user", "content": "Hello"}]
  }'
```

### 2. 性能基准测试
```bash
# 高负载性能测试
./bin/llm-d-inference-sim \
  --model "benchmark-model" \
  --mode random \
  --time-to-first-token 2000 \
  --inter-token-latency 100 \
  --max-num-seqs 10 \
  --data-parallel-size 8
```

### 3. 集成测试
```bash
# 与llm-d生态系统集成测试
kubectl apply -f manifests/deployment.yaml

# 验证服务
kubectl get deployment inference-sim
kubectl get service inference-sim

# 端口转发测试
kubectl port-forward svc/inference-sim 8000:8000
```

## 性能特性

### 资源效率
- **CPU-only**: 无需GPU资源，适合测试环境
- **轻量级**: 最小内存占用，支持大规模并发测试
- **可配置**: 精确的性能特征模拟

### 高级功能
- **数据集支持**: 基于SQLite的真实数据集响应
- **故障注入**: 测试系统的容错能力
- **KV缓存**: 模拟真实的缓存行为和性能

### 扩展性
- **多实例**: 支持数据并行多实例部署
- **动态配置**: 支持运行时配置更新
- **插件化**: 可扩展的自定义响应生成

## 监控和可观测性

### 指标监控
```bash
# 访问Prometheus指标
curl http://localhost:8000/metrics

# 关键指标示例
vllm:gpu_cache_usage_perc 0.0
vllm:num_requests_running 3
vllm:num_requests_waiting 5
vllm:e2e_request_latency_seconds_bucket{le="1.0"} 100
vllm:time_to_first_token_seconds_bucket{le="2.0"} 150
```

### 日志分析
```bash
# 查看详细日志
./bin/llm-d-inference-sim --v=2 --logtostderr=true

# 结构化日志输出
{
  "level": "info",
  "timestamp": "2025-12-04T11:00:00Z",
  "message": "Request processed",
  "request_id": "req-123",
  "model": "test-model",
  "response_time_ms": 1500
}
```

### 健康检查
```bash
# 健康状态检查
curl http://localhost:8000/health

# 就绪状态检查  
curl http://localhost:8000/ready
```

## 故障排查

### 常见问题
```yaml
# 端口占用错误
错误: "bind: address already in use"
解决: 更改端口或停止占用进程

# 配置语法错误
错误: "failed to parse lora-modules"
解决: 检查JSON格式和转义字符

# 内存不足
错误: "cannot allocate memory"
解决: 减少max-num-seqs或增加内存限制

# 网络连接失败
错误: "failed to publish to ZMQ endpoint"
解决: 检查ZMQ地址配置和防火墙设置
```

### 调试方法
```bash
# 启用详细日志
./bin/llm-d-inference-sim --v=5 --logtostderr=true

# 使用调试模式
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "test", "messages": [{"role": "user", "content": "debug"}]}' \
  -v

# 检查进程状态
ps aux | grep inference-sim
netstat -tlnp | grep :8000
```

## 版本演进

### v0.6.1 当前特性
- 支持SQLite数据集集成
- 改进故障注入机制
- 优化内存使用
- 增强错误处理

### 从v0.2.0迁移
```bash
# 参数变更对照
旧参数: max-running-requests → 新参数: max-num-seqs
旧参数: lora → 新参数: lora-modules (JSON数组格式)
```

## 最佳实践

### 测试环境使用
1. **性能测试**: 使用精确的延迟参数模拟真实场景
2. **压力测试**: 配置高并发数和队列长度
3. **故障测试**: 启用故障注入验证系统弹性
4. **集成测试**: 使用与生产相同的API接口

### 生产环境预检
1. **API兼容性**: 验证客户端与vLLM API兼容性
2. **负载测试**: 使用模拟器测试系统负载能力
3. **监控集成**: 验证监控指标和告警配置
4. **故障演练**: 模拟各种故障场景测试恢复能力

### 开发工作流
1. **本地开发**: 快速启动模拟器进行功能开发
2. **CI/CD集成**: 在测试流水线中使用模拟器
3. **性能回归**: 定期运行基准测试监控性能
4. **文档测试**: 使用模拟器验证API文档示例

## 与llm-d生态集成

### 与Inference Scheduler配合
- **端点注册**: 模拟器可作为EPP的后端端点
- **指标上报**: 提供完整的Prometheus指标
- **健康检查**: 支持标准健康检查机制

### 与Model Service配合
- **LoRA管理**: 支持动态LoRA适配器管理
- **模型切换**: 支持多模型部署和切换
- **资源管理**: 精确控制资源使用

### 与Benchmark工具配合
- **负载生成**: 可作为llm-d-benchmark的测试目标
- **性能数据**: 提供可重现的性能基准
- **场景模拟**: 支持多种工作负载场景
