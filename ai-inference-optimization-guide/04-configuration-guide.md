# vLLM 配置参数详细指南

## 1. 配置参数分类概览

### 1.1 参数分类体系
```
┌─────────────────────────────────────────────────────────────┐
│                    配置参数分类                               │
├─────────────────────────────────────────────────────────────┤
│ 🔴 关键参数: 性能影响巨大，必须正确配置                     │
│ ⚡ 性能参数: 优化吞吐量和延迟                                │
│ 🏗️ 架构参数: 控制并行和分布式部署                            │
│ 💾 内存参数: 管理GPU内存使用                                │
│ 🎛️ 调度参数: 控制请求调度和批处理                            │
│ 🔧 调试参数: 开发和调试专用                                  │
│ 🌐 网络参数: 分布式通信配置                                  │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 配置优先级矩阵
```
┌─────────────────┬─────────────────┬─────────────────┐
│                 │ 高影响          │ 中影响          │
├─────────────────┼─────────────────┼─────────────────┤
│ 基础配置        │ enforce-eager   │ tensor-parallel │
│                 │ model-path      │ max-model-len   │
├─────────────────┼─────────────────┼─────────────────┤
│ 性能优化        │ dtype          │ async-scheduling│
│                 │ kv-cache-dtype │ block-size      │
├─────────────────┼─────────────────┼─────────────────┤
│ 高级调优        │ max-num-seqs   │ swap-space      │
│                 │ chunked-prefill│ scheduling-policy│
└─────────────────┴─────────────────┴─────────────────┘
```

## 2. 🔴 关键参数详解

### 2.1 enforce-eager (性能杀手参数)

#### 2.1.1 参数说明
```bash
--enforce-eager  # 强制使用eager模式，禁用CUDA Graph
```

#### 2.1.2 性能影响分析
```
对比数据 (Test 4 vs Test 5):
- 启用eager: 777 tok/s, 6.43s延迟
- 禁用eager: 1966 tok/s, 2.62s延迟
- 性能差距: -153% 吞吐量, +59% 延迟

根本原因:
- 每个操作独立Kernel启动
- 运行时内存分配开销
- 无法利用计算图优化
- 串行执行限制并行度
```

#### 2.1.3 配置建议
```
✅ 推荐配置: 不使用此参数 (默认开启CUDA Graph)
❌ 避免场景: 生产环境、性能敏感服务
⚠️  特殊用途: 调试阶段、需要逐步执行的场景

回滚检查:
grep "enforce-eager" 启动脚本
```

### 2.2 基础模型参数

#### 2.2.1 模型路径与名称
```bash
--model /path/to/model              # 模型权重路径
--served-model-name model-name      # 服务显示的模型名称
--trust-remote-code                 # 信任远程代码 (特殊模型需要)
```

#### 2.2.2 网络配置
```bash
--host 0.0.0.0                     # 监听地址
--port 8200                        # 服务端口
```

## 3. ⚡ 性能优化参数

### 3.1 数据类型配置

#### 3.1.1 dtype (推理精度)
```bash
--dtype bfloat16                    # 推荐配置 (B200硬件)
--dtype float16                     # 默认配置
--dtype float32                     # 高精度但低性能
```

**精度对比分析**:
```
┌─────────────┬──────────┬──────────┬──────────┐
│ 数据类型    │ 内存占用 │ 计算速度 │ 数值稳定性│
├─────────────┼──────────┼──────────┼──────────┤
│ float32     │ 100%     │ 基准     │ 最高     │
│ float16     │ 50%      │ +30%     │ 中等     │
│ bfloat16    │ 50%      │ +35%     │ 较高     │
│ fp8         │ 25%      │ +50%     │ 较低     │
└─────────────┴──────────┴──────────┴──────────┘
```

**推荐配置**:
```bash
# B200/A100+ 硬件
--dtype bfloat16 --kv-cache-dtype "fp8"

# 较老硬件
--dtype float16
```

#### 3.1.2 kv-cache-dtype (KV缓存量化)
```bash
--kv-cache-dtype "fp8"             # 8位量化 (推荐)
--kv-cache-dtype "fp8_e5m2"        # FP8变体
--kv-cache-dtype "auto"            # 自动选择
```

**FP8量化效果**:
```
内存节省: 50% (FP16→FP8)
性能影响: +5-10% 计算开销
精度损失: 微小 (主要影响长文本一致性)
适用场景: 高并发、长文本、内存受限环境
```

### 3.2 内存管理参数

#### 3.2.1 GPU内存利用率
```bash
--gpu-memory-utilization 0.90       # 推荐配置 (激进但安全)
--gpu-memory-utilization 0.85       # 保守配置
--gpu-memory-utilization 0.95       # 极限配置 (OOM风险)
```

**调优策略**:
```
利用率区间建议:
0.80-0.85: 稳定性优先，生产环境初始配置
0.86-0.90: 性能均衡，推荐配置
0.91-0.95: 性能优先，需要充分测试
>0.95: 极限性能，高风险配置

计算公式:
实际使用 = GPU总内存 × 利用率 - 模型权重 - 系统预留
```

#### 3.2.2 模型长度配置
```bash
--max-model-len 262144             # 最大上下文长度
--max-num-batched-tokens 262144    # 批处理tokens数
```

**配置建议**:
```
计算方法:
max_len = floor((可用内存 - 模型权重) / 
               (2 × 隐藏层大小 × 层数 × token_size))

推荐配置:
- max-num-batched-tokens = max-model-len
- 留15-20%安全边际
- 基于实际prompt长度统计设置
```

## 4. 🏗️ 架构参数详解

### 4.1 并行策略配置

#### 4.1.1 张量并行
```bash
--tensor-parallel-size 4            # 4路张量并行
--tensor-parallel-size 8            # 8路张量并行 (适合8卡)
```

**并行策略分析**:
```
当前环境 (B200×8):
- 4路并行: 每卡2个模型副本，隐式流水线
- 8路并行: 每卡1个模型副本，最大并行度

性能权衡:
- 4路: 通信少，计算密集，内存分布均匀
- 8路: 并行度高，通信开销大，内存利用率高

选择建议:
- 小模型 (<50B): 8路并行
- 大模型 (100B+): 4路并行
- 当前235B模型: 4路并行合适
```

#### 4.1.2 分布式配置
```bash
--distributed-executor-backend ray # Ray分布式后端
--worker-use-ray                   # Worker使用Ray
```

### 4.2 GPU设备配置
```bash
CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7  # GPU设备选择
```

## 5. 🎛️ 调度与批处理参数

### 5.1 并发控制参数

#### 5.1.1 最大并发数
```bash
--max-num-seqs 64                   # 默认配置
--max-num-seqs 96                   # 推荐优化配置
--max-num-seqs 128                  # 高并发配置
```

**调优策略**:
```
并发数选择原则:
- 基于GPU内存容量
- 考虑延迟要求
- 监控排队效应

调优步骤:
1. 从64开始
2. 渐进增加到96、112、128
3. 监控P95延迟变化
4. 观察GPU内存使用率

性能影响:
- 64→96: +10-15% 吞吐量
- 64→128: +15-25% 吞吐量，P95延迟可能增加
```

#### 5.1.2 请求调度策略
```bash
--scheduling-policy fcfs            # 先来先服务 (默认)
--scheduling-policy priority        # 优先级调度
```

**策略对比**:
```
FCFS (First Come First Served):
✅ 公平性好，无饥饿现象
✅ 实现简单，调试容易
❌ 长请求阻塞短请求

Priority-based:
✅ 重要请求优先处理
✅ 支持SLA差异化
❌ 实现复杂，需要优先级定义
```

### 5.2 批处理优化参数

#### 5.2.1 Chunked Prefill
```bash
--enable-chunked-prefill           # 启用分块预填充
```

**优化机制**:
```
传统Prefill: 等待完整prompt → 一次性处理 → 延迟高
Chunked Prefill: 分块处理 → 流式预填充 → 延迟低

适用场景:
✅ 长prompt (>512 tokens)
✅ 高并发请求
✅ 实时性要求高

不适用场景:
❌ 短prompt (<64 tokens)
❌ 低并发场景
❌ 离线批处理
```

#### 5.2.2 动态批处理
```bash
--max-num-batched-tokens 262144     # 批处理大小
```

**批处理效果**:
```
批次大小影响:
- GPU利用率: 随批次大小提升
- 内存需求: 线性增长
- 首token延迟: 可能增加

调优原则:
- 高吞吐场景: 最大化批次大小
- 低延迟场景: 适度批次大小
- 平衡场景: 根据SLA要求调整
```

## 6. 🔧 调试与监控参数

### 6.1 调试参数
```bash
--disable-log-requests             # 禁用请求日志
--disable-log-stats                 # 禁用统计日志
--log-level debug                  # 调试日志级别
--disable-frontend-multiprocessing  # 禁用前端多进程
```

### 6.2 监控参数
```bash
--disable-slots                   # 禁用slot机制 (调试用)
--quantization fp8                 # FP8量化 (实验性)
```

## 7. 🌐 网络与分布式参数

### 7.1 NCCL配置
```bash
NCCL_DEBUG=INFO                    # NCCL调试信息
NCCL_SOCKET_IFNAME=eth0           # 网络接口
NCCL_IB_DISABLE=0                 # 启用InfiniBand
```

### 7.2 Ray分布式配置
```bash
--ray-address ray-head:6379        # Ray集群地址
--ray-namespace vllm              # Ray命名空间
```

## 8. 完整配置示例

### 8.1 🚀 最优生产配置 (基于测试验证)
```bash
python3 -m vllm.entrypoints.openai.api_server \
  --model /model-cache/Qwen/Qwen3-VL-235B-A22B-Instruct \
  --served-model-name Qwen3-VL-235B-A22B-Instruct \
  --host 0.0.0.0 \
  --port 8200 \
  --tensor-parallel-size 4 \
  --max-model-len 262144 \
  --enable-chunked-prefill \
  --max-num-batched-tokens 262144 \
  --gpu-memory-utilization 0.90 \
  --max-num-seqs 96 \
  --dtype bfloat16 \
  --kv-cache-dtype "fp8" \
  --trust-remote-code
  # 关键: 不要添加 --enforce-eager
```

### 8.2 🎯 高并发配置
```bash
# 在最优配置基础上
  --max-num-seqs 128 \
  --async-scheduling \
  --scheduling-policy fcfs
```

### 8.3 🛡️ 保守稳定配置
```bash
# 在最优配置基础上
  --gpu-memory-utilization 0.85 \
  --max-num-seqs 64
```

### 8.4 🧪 调试配置
```bash
# 添加调试参数
  --enforce-eager \
  --disable-log-requests \
  --log-level debug
```

## 9. 配置验证检查清单

### 9.1 启动前检查
```
□ 模型路径正确且可访问
□ GPU设备数量匹配配置
□ 内存利用率设置合理 (0.80-0.95)
□ 不包含 --enforce-eager (生产环境)
□ 并行策略与硬件匹配
□ 网络配置正确 (分布式环境)
```

### 9.2 启动后验证
```
□ CUDA Graph已启用 (日志确认)
□ GPU内存使用正常
□ 服务响应正常
□ 并发处理能力符合预期
□ 监控指标正常
```

### 9.3 性能基准
```
□ 吞吐量 >1500 tok/s
□ P50延迟 <3s
□ P95延迟 <10s
□ GPU利用率 >80%
□ 无OOM错误
```

## 10. 常见配置错误

### 10.1 性能杀手配置
```bash
# ❌ 错误配置
--enforce-eager                    # 禁用CUDA Graph
--gpu-memory-utilization 1.0       # OOM风险
--max-num-seqs 256                  # 过度并发
--dtype float32                    # 低效精度
```

### 10.2 硬件不匹配配置
```bash
# ❌ 错误配置 (8卡用16路并行)
--tensor-parallel-size 16

# ❌ 错误配置 (GPU内存不足)
--max-model-len 1000000

# ❌ 错误配置 (不支持FP8的硬件)
--kv-cache-dtype "fp8"
```

---

## 工程师配置要点

1. **优先检查**: 确保`--enforce-eager`参数不存在
2. **渐进调优**: 从保守配置开始，逐步优化
3. **硬件匹配**: 并行策略必须与GPU数量匹配
4. **内存安全**: GPU利用率不超过95%，留安全边际
5. **验证优先**: 每次配置变更都要进行性能验证

这个配置指南提供了完整的参数说明和实用建议，可以作为日常配置的参考手册。
