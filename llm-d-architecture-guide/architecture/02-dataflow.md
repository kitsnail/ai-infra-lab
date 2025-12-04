# llm-d 数据流架构

## 请求处理流程

### 智能推理调度模式（v1.3.1验证）

```
客户端请求 → Istio Gateway → EnvoyFilter(BBR) → HTTPRoute → EPP → Routing Sidecar → vLLM/SGLang → 响应
    ↓              ↓               ↓              ↓        ↓             ↓            ↓
  (HTTP 80)     (Body解析)    (Model匹配)    (调度决策) (NIXLv2路由)   (TP8/FP8)   (结果返回)
```

**实际配置链**:
```yaml
# EnvoyFilter v2 链式处理
- BBR (INSERT_FIRST): FULL_DUPLEX_STREAMED 模式
- EPP (INSERT_AFTER): BUFFERED 模式
```

### 预填充/解码分离模式

```
客户端请求 → Gateway → Prefill服务器 → KV缓存生成 → Sidecar(NIXL) → Decode服务器 → 响应
                                       ↓
                                   高速互连传输
```

### 预填充/解码分离模式

```
客户端请求 → Gateway → Prefill服务器 → KV缓存生成 → Sidecar(NIXL) → Decode服务器 → 响应
                                       ↓
                                   高速互连传输
```

### 宽专家并行模式

```
客户端请求 → Gateway → Expert Router → 专家节点池 → 结果聚合 → 响应
                    ↓
               专家选择策略
```

## 核心数据路径

### 1. 请求入口层
- **Gateway**: 基于Istio或原生K8s Gateway
- **HTTPRoute**: 路由规则匹配
- **Body-Based Router**: 请求体解析和头部注入

### 2. 路由决策层
- **Endpoint Picker (EPP)**: 端点选择核心
- **调度插件**: 过滤器 + 评分器
- **缓存感知**: KV缓存状态查询

### 3. 推理执行层
- **vLLM Engine**: 模型服务引擎
- **KVConnector**: 缓存管理接口
- **Prefill/Decode角色**: 处理阶段分离

### 4. 数据存储层（v1.3.1验证）
- **本地缓存**: 内存/磁盘KV缓存 + 64Gi共享内存优化
- **共享缓存**: 跨实例缓存索引
- **持久存储**: PVC + ModelScope 模型管理
- **ModelScope集成**: 大规模模型下载和版本管理

**实际配置示例**:
```yaml
# DeepSeek V3.1 存储配置
modelArtifacts:
  uri: "pvc://llm-model/deepseek-ai/DeepSeek-V3.1"
  name: DeepSeek-V3.1

volumes:
  - name: shm
    emptyDir:
      medium: Memory
      sizeLimit: 64Gi # 共享内存优化
```

## 关键交互点

### EPP与vLLM交互
```yaml
# 监控指标采集
metrics:
  - vllm:num_requests_waiting    # 队列深度
  - vllm:gpu_cache_usage_perc    # GPU缓存利用率
  - vllm:prompt_tokens           # 前缀令牌数
```

### KV缓存传输
```yaml
# NIXL配置
transmission:
  protocol: RDMA/RoCE/TPU ICI
  compression: true
  encryption: optional
```

### Gateway路由配置
```yaml
# HTTPRoute匹配规则
matches:
  headers:
    name: X-Gateway-Model-Name
    type: Exact
    value: "Model-Name"
```

## 性能关键路径

### 延迟敏感路径
1. **Cold Start**: 模型加载和初始化
2. **Prefill Stage**: 提示处理和KV生成
3. **First Token**: 首个令牌生成时间(TTFT)
4. **Token Generation**: 令牌生成吞吐量(TPOT)

### 优化策略
- **预填充缓存**: 重用已计算的KV状态
- **负载均衡**: 基于队列深度和GPU利用率
- **路径优化**: 高速互连直接传输
- **批处理**: 请求合并处理

## 扩展性设计

### 水平扩展
- **Pod副本**: 基于HPA自动扩缩容
- **节点扩展**: 集群容量动态调整
- **缓存分片**: 分布式KV缓存管理

### 垂直扩展
- **GPU配额**: 动态GPU资源分配
- **内存优化**: KV缓存内存管理
- **网络带宽**: 高速互连带宽分配

## 故障处理机制

### 容错策略
- **端点健康检查**: 实时状态监控
- **自动故障转移**: 失败端点自动排除
- **缓存恢复**: 损坏缓存自动重建

### 降级方案
- **缓存绕过**: 缓存失败直接计算
- **简化路由**: 降级到轮询负载均衡
- **资源隔离**: 故障节点隔离

## 监控可观测性

### 关键指标
```yaml
performance_metrics:
  - request_latency_p50/p90/p99
  - throughput_tokens_per_second
  - cache_hit_ratio
  - gpu_utilization
  - queue_depth

operational_metrics:
  - pod_restart_count
  - error_rate
  - oom_kills
  - network_latency
```

### 日志策略
- **结构化日志**: JSON格式统一输出
- **请求追踪**: X-Request-ID全链路追踪
- **性能剖析**: 关键路径时间分析
