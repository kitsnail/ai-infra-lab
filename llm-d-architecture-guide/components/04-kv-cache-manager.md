# KV Cache Manager 组件详解

## 核心功能

KV Cache Manager是llm-d的核心缓存管理组件，支持KV缓存感知路由和跨节点缓存协调，为vLLM服务提供高级缓存管理基础。

**关键优势**: 分层存储架构，分布式缓存一致性

## 缓存架构

### 分层存储
| 层级 | 类型 | 访问速度 | 容量配比 |
|------|------|----------|----------|
| **L1** | 本地内存 | 最高 | 80% |
| **L2** | 本地SSD | 高 | 15% |
| **L3** | 共享存储 | 中 | 5% |

### 核心特性
- **前缀缓存**: 基于提示前缀的智能匹配
- **跨节点协调**: 全局缓存索引和同步
- **可插拔接口**: 通过KVConnector集成vLLM
- **一致性保证**: 分布式缓存一致性机制

## 技术配置

### KVConnector接口
```yaml
# 可插拔缓存接口
kvconnector:
  type: tiered_prefix_cache
  features:
    - prefix_hashing
    - lru_eviction
    - cross_node_sharing
    - compression
    
  storage:
    local:
      type: memory
      max_size: "100GB"
      
    disk:
      type: nvme
      path: "/tmp/kv_cache"
      max_size: "1TB"
      
    shared:
      type: s3
      bucket: "llm-d-kv-cache"
      compression: true
```

### 前缀缓存配置 (v1.3.1验证)
```yaml
prefix_cache:
  enabled: true
  block_size: 64
  max_blocks: 100000
  hash_algorithm: "xxhash"
  
# EPP集成
prefix_cache_scorer:
  type: prefix-cache-scorer
  parameters:
    hash_block_size: 64
    max_prefix_blocks_to_match: 256
    lru_capacity_per_server: 31250
    cache_hit_bonus: 100
```

### 实际配置示例
```yaml
# DeepSeek V3.1 缓存优化
memory:
  - name: shm
    emptyDir:
      medium: Memory
      sizeLimit: 64Gi # 增加共享内存优化

# Qwen3-VL-235B KV缓存
args:
  - --kv-cache-datatype "fp8"
  - --enable-prefix-caching
  - --gpu-memory-utilization 0.85
  - --max-num-batched-tokens 26214
```

## 性能优化

### LRU策略
```yaml
lru_policy:
  type: weighted_lru
  weights:
    - metric: access_frequency
      weight: 0.5
    - metric: cache_size  
      weight: 0.3
    - metric: access_time
      weight: 0.2

eviction:
  threshold: 0.9  # 90%使用率触发淘汰
  strategy: least_recently_used
  batch_size: 1000
```

### 内存管理
```yaml
memory:
  gc_threshold: 0.85
  gc_interval: "60s"
  max_concurrent_gc: 4
  
network:
  transfer_timeout: "30s"
  retry_count: 3
  bandwidth_limit: "1Gbps"
  
concurrency:
  max_concurrent_transfers: 10
  queue_size: 1000
  worker_threads: 8
```

## 集成配置

### vLLM集成
```yaml
vllm_config:
  kv_connector_config:
    type: "tiered"
    local_path: "/tmp/local_cache"
    remote_path: "s3://llm-d-cache/"
    compression: true
    
  cache_config:
    gpu_memory_fraction: 0.8
    cpu_swap_size: "64GB"
    enable_caching: true
```

### 压缩配置
```yaml
compression:
  enabled: true
  algorithm: "lz4"
  level: 6
  threshold_size: "1MB"
  
sync:
  enabled: true
  interval: "30s"
  consistency_level: "eventual"
```

## 监控指标

### 缓存性能
```yaml
metrics:
  - cache_hit_ratio
  - cache_miss_ratio
  - cache_eviction_rate
  - cache_utilization
  - cache_lookup_latency
  - cache_storage_latency
  - cache_transfer_bandwidth
  - cache_size_bytes
  - cache_items_count
  - cache_memory_usage
  - cache_disk_usage
```

### 分布式指标
```yaml
distributed_metrics:
  - sync_status
  - node_cache_consistency
  - cross_node_transfers
  - global_index_size
```

## 故障排查

### 常见问题
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **缓存命中率低** | 块大小不匹配或工作负载特征不符 | 调整block_size和容量配置 |
| **内存使用过高** | LRU策略配置错误或缓存过大 | 调整淘汰阈值和容量限制 |
| **跨节点同步失败** | 网络连接问题或权限配置错误 | 检查网络连通性和IAM权限 |
| **磁盘空间不足** | 缓存淘汰策略失效 | 检查GC配置和磁盘清理策略 |

### 调试命令
```bash
# 检查缓存状态
kubectl exec -it <kv-cache-pod> -- curl localhost:8080/cache/status

# 监控指标
kubectl port-forward <kv-cache-pod> 9090:9090
curl http://localhost:9090/metrics | grep cache_hit_ratio

# 查看缓存日志
kubectl logs -f <kv-cache-pod> | grep -E "(cache|evict|sync)"
```

## 工作负载优化

### 基于工作负载的配置
```yaml
workload_profiles:
  chat_completions:
    block_size: 64
    max_blocks: 50000
    
  code_completion:
    block_size: 128
    max_blocks: 100000
    
  summarization:
    block_size: 32
    max_blocks: 25000
```

### 存储优化
```yaml
storage_optimization:
  memory_cache:
    priority: high
    eviction_policy: lru
    
  disk_cache:
    priority: medium
    compression: true
    encryption: false
    
  remote_cache:
    priority: low
    async_upload: true
    bandwidth_limit: "500Mbps"
```

## 版本信息

- **当前版本**: v0.3.0
- **核心特性**:
  - 分层缓存架构
  - 跨节点协调
  - 前缀缓存
  - 压缩和一致性

## 最佳实践

### 生产部署
1. **容量规划**: 根据模型大小和请求模式合理配置缓存容量
2. **存储选择**: 高速SSD用于本地缓存，网络存储用于共享
3. **监控告警**: 设置关键指标的阈值告警
4. **备份策略**: 定期备份重要缓存数据

### 性能调优
1. **块大小**: 根据平均提示长度调整块大小
2. **淘汰策略**: 选择适合工作负载的LRU变体
3. **压缩配置**: 平衡压缩率和CPU开销
4. **网络优化**: 优化跨节点传输参数

### 运维管理
1. **定期清理**: 定期清理过期和损坏的缓存
2. **容量监控**: 监控存储使用情况及时扩容
3. **性能基准**: 定期进行缓存性能基准测试
4. **故障恢复**: 建立缓存故障恢复机制
