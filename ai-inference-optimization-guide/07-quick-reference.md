# 快速参考手册与索引

## 1. 快速参考清单

### 1.1 🚀 立即行动清单

#### 关键优化（必须做）
```bash
# 检查并移除性能杀手参数
grep "enforce-eager" 启动脚本  # 如果有，立即删除

# 启用最优配置
--dtype bfloat16 --kv-cache-dtype "fp8"

# 调整内存利用率
--gpu-memory-utilization 0.90
```

#### 基础验证（快速检查）
```bash
# 服务状态检查
curl -f http://localhost:8200/health

# 性能基准测试
curl -X POST http://localhost:8200/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen3-VL-235B-A22B-Instruct", "prompt": "测试", "max_tokens": 50}' --time

# GPU状态检查
nvidia-smi --query-gpu=utilization.gpu,utilization.memory,memory.used,memory.total,temperature.gpu --format=csv
```

### 1.2 📊 性能指标速查表

#### 基准性能（B200×8 + Qwen3-VL-235B）
```
┌─────────────────┬──────────┬──────────┬──────────┐
│ 配置级别       │ 吞吐量   │ P50延迟  │ P95延迟  │
├─────────────────┼──────────┼──────────┼──────────┤
│ 差配置         │ ~800     │ 5.2s     │ 15s      │
│ 基线配置       │ ~1200    │ 4.0s     │ 12s      │
│ 优化配置       │ ~2000    │ 1.4s     │ 8s       │
│ 极致配置       │ ~2500    │ 1.0s     │ 6s       │
└─────────────────┴──────────┴──────────┴──────────┘
```

#### 告警阈值（快速判断）
```
正常:     吞吐量>1800 tok/s, P50<2s, GPU>80%
警告:     吞吐量下降>20%, P50>4s, GPU<70%
严重:     吞吐量下降>50%, P50>8s, OOM错误
```

### 1.3 🛠️ 配置模板速查

#### 生产环境配置
```bash
python3 -m vllm.entrypoints.openai.api_server \
  --model /model-cache/Qwen/Qwen3-VL-235B-A22B-Instruct \
  --served-model-name Qwen3-VL-235B-A22B-Instruct \
  --host 0.0.0.0 --port 8200 \
  --tensor-parallel-size 4 \
  --max-model-len 262144 \
  --enable-chunked-prefill \
  --max-num-batched-tokens 262144 \
  --gpu-memory-utilization 0.90 \
  --max-num-seqs 96 \
  --dtype bfloat16 \
  --kv-cache-dtype "fp8" \
  --trust-remote-code
  # 关键：不要加 --enforce-eager
```

#### 开发环境配置
```bash
# 在生产配置基础上修改：
--gpu-memory-utilization 0.80 \
--max-num-seqs 32 \
--log-level DEBUG \
# 调试时可加：--enforce-eager
```

#### 高延迟场景配置
```bash
# 在生产配置基础上修改：
--max-num-seqs 32 \
--max-num-batched-tokens 65536 \
--gpu-memory-utilization 0.85
```

## 2. 故障排除速查表

### 2.1 🚨 紧急故障快速响应

#### OOM故障
```bash
# 立即措施
1. 降低并发：--max-num-seqs 32
2. 降低内存：--gpu-memory-utilization 0.80
3. 重启服务：pkill -f vllm
4. 清理GPU：nvidia-smi --gpu-reset
```

#### 性能骤降
```bash
# 检查清单
□ 配置中是否包含 --enforce-eager
□ GPU是否降频（温度过高）
□ 网络延迟是否异常
□ 模型文件是否完整
```

#### 服务不可用
```bash
# 快速诊断
netstat -tlnp | grep 8200  # 端口检查
ps aux | grep vllm           # 进程检查
nvidia-smi                   # GPU检查
curl http://localhost:8200/health  # 健康检查
```

### 2.2 🔧 常见问题快速解决

| 问题 | 快速诊断 | 解决方案 |
|------|----------|----------|
| 吞吐量低 | 检查--enforce-eager参数 | 移除该参数 |
| 延迟高 | 检查max-num-seqs设置 | 适当减少或增加 |
| OOM频繁 | 检查内存利用率设置 | 降低到0.85 |
| GPU利用率低 | 检查批处理大小 | 增加max-num-batched-tokens |
| 启动失败 | 检查模型路径和权限 | 确认路径正确和权限足够 |

### 2.3 📈 性能调优速查

#### 吞吐量优化
```
正向：增加max-num-seqs, 增加batched-tokens, 启用FP8
负向：过大的并发可能导致排队，需要平衡
```

#### 延迟优化
```
TTFT：启用chunked-prefill, 减少batch size
TPOT：提升GPU利用率, 启用CUDA Graph
P95：优化并发参数, 避免资源竞争
```

#### 成本优化
```
内存：启用FP8量化, 提升GPU利用率
硬件：合理并行策略, 智能调度
运维：自动化监控, 预防性维护
```

## 3. 参数配置速查表

### 3.1 🔴 关键参数影响矩阵

| 参数 | 影响 | 推荐值 | 错误值 | 性能影响 |
|------|------|--------|--------|----------|
| enforce-eager | CUDA Graph | 不使用 | 使用 | -153% |
| gpu-memory-utilization | 内存使用 | 0.90 | >0.95 | OOM风险 |
| dtype | 计算精度 | bfloat16 | float32 | -50% |
| kv-cache-dtype | 内存占用 | fp8 | fp16 | +100%内存 |
| max-num-seqs | 并发数 | 96 | 32 | -20%吞吐 |
| tensor-parallel-size | 并行度 | 4 | 1 | -75% |

### 3.2 ⚡ 优化参数组合

#### 吞吐量优先
```bash
--gpu-memory-utilization 0.95 \
--max-num-seqs 128 \
--max-num-batched-tokens 524288 \
--kv-cache-dtype "fp8" \
--async-scheduling
```

#### 延迟优先
```bash
--max-num-seqs 32 \
--max-num-batched-tokens 65536 \
--enable-chunked-prefill \
--gpu-memory-utilization 0.85
```

#### 稳定性优先
```bash
--gpu-memory-utilization 0.80 \
--max-num-seqs 64 \
--kv-cache-dtype "auto" \
--log-level INFO
```

### 3.3 🎛️ 硬件适配配置

#### B200/A100（现代GPU）
```bash
--dtype bfloat16 \
--kv-cache-dtype "fp8" \
--tensor-parallel-size 4 \
--gpu-memory-utilization 0.90-0.95
```

#### V100/T4（老旧GPU）
```bash
--dtype float16 \
--kv-cache-dtype "auto" \
--tensor-parallel-size 2 \
--gpu-memory-utilization 0.80-0.85
```

## 4. 监控指标速查

### 4.1 📊 业务指标

| 指标 | 正常范围 | 警告 | 严重 |
|------|----------|-------|------|
| 吞吐量 | >1800 tok/s | <1500 | <1000 |
| P50延迟 | <2s | >3s | >5s |
| P95延迟 | <8s | >12s | >20s |
| 成功率 | >99% | <98% | <95% |

### 4.2 🔧 系统指标

| 指标 | 正常范围 | 警告 | 严重 |
|------|----------|-------|------|
| GPU利用率 | 80-95% | 70-80% | <70% |
| GPU内存 | 70-90% | >95% | 100% |
| GPU温度 | <75°C | 75-85°C | >85°C |
| CPU利用率 | 60-80% | >90% | >95% |

### 4.3 🚨 告警速查

#### Critical告警（立即响应）
```
- 服务不可用（5xx错误 >5%）
- GPU内存OOM
- 吞吐量下降 >50%
- P95延迟 >15s
```

#### Warning告警（1小时内响应）
```
- 吞吐量下降 >20%
- GPU利用率 <70%
- P50延迟 >4s
- 错误率 >3%
```

## 5. 命令行工具速查

### 5.1 🔍 诊断命令

```bash
# 基础诊断
nvidia-smi                          # GPU状态
ps aux | grep vllm                 # 进程状态
netstat -tlnp | grep 8200          # 端口状态
free -h                           # 内存使用
df -h /model-cache                 # 磁盘空间

# 性能测试
curl -w "@curl-format.txt" -o /dev/null -s "http://localhost:8200/health"
stress --cpu 4 --timeout 60s        # CPU压力测试
```

### 5.2 📈 监控命令

```bash
# 实时监控
watch -n 1 'nvidia-smi --query-gpu=utilization.gpu,utilization.memory,temperature.gpu --format=csv,noheader'
tail -f /var/log/vllm/server.log

# 网络监控
iftop -i eth0                    # 网络流量
ss -tulnp | grep 8200              # 连接状态

# 系统资源
htop                             # 进程监控
iotop                            # IO监控
```

### 5.3 🛠️ 维护命令

```bash
# 服务管理
systemctl status vllm               # 服务状态
systemctl restart vllm             # 重启服务
systemctl reload vllm              # 重载配置

# 清理操作
pkill -f vllm                    # 强制停止
nvidia-smi --gpu-reset            # 重置GPU
rm -rf /tmp/vllm-*              # 清理临时文件

# 日志管理
journalctl -u vllm -f            # 查看日志
logrotate -f /etc/logrotate.d/vllm # 轮转日志
```

## 6. 配置文件速查

### 6.1 📋 环境变量配置

```bash
# 必需环境变量
export CUDA_VISIBLE_DEVICES=0,1,2,3
export NCCL_DEBUG=INFO
export NCCL_SOCKET_IFNAME=eth0

# 可选优化
export OMP_NUM_THREADS=8
export CUDA_LAUNCH_BLOCKING=0
export PYTHONPATH=$PYTHONPATH:/opt/vllm
```

### 6.2 🐳 Docker配置

```yaml
# 生产环境Docker Compose
version: '3.8'
services:
  vllm:
    image: vllm/vllm-openai:latest
    environment:
      - CUDA_VISIBLE_DEVICES=0,1,2,3
    volumes:
      - /model-cache:/model-cache
    ports:
      - "8200:8200"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 4
              capabilities: [gpu]
```

### 6.3 ⚙️ Nginx配置

```nginx
upstream vllm_backend {
    server 127.0.0.1:8200 max_fails=3 fail_timeout=30s;
}

server {
    listen 443 ssl http2;
    
    location / {
        proxy_pass http://vllm_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
    
    location /health {
        proxy_pass http://vllm_backend/health;
        access_log off;
    }
}
```

## 7. 文档索引

### 7.1 📚 核心文档导航

```
ai-inference-optimization-guide/
├── README.md                     # 总体介绍
├── 01-architecture-overview.md    # 架构概述
├── 02-performance-analysis.md     # 性能分析
├── 03-optimization-strategy.md    # 优化策略
├── 04-configuration-guide.md     # 配置指南
├── 05-troubleshooting.md         # 故障排除
├── 06-best-practices.md         # 最佳实践
├── 07-quick-reference.md         # 快速参考（本文档）
└── data/
    └── test-results.md           # 测试数据
```

### 7.2 🎯 按角色查找信息

#### 架构师
- 系统设计：01-architecture-overview.md
- 性能规划：02-performance-analysis.md
- 优化策略：03-optimization-strategy.md

#### 工程师
- 配置参数：04-configuration-guide.md
- 故障处理：05-troubleshooting.md
- 最佳实践：06-best-practices.md

#### 运维工程师
- 监控体系：05-troubleshooting.md
- 运维流程：06-best-practices.md
- 快速参考：07-quick-reference.md

### 7.3 🏷️ 按问题类型查找

#### 性能问题
- 吞吐量低：02-performance-analysis.md#性能瓶颈分析
- 延迟高：02-performance-analysis.md#延迟优化
- OOM问题：05-troubleshooting.md#oom故障

#### 配置问题
- 参数说明：04-configuration-guide.md
- 环境配置：06-best-practices.md#配置管理
- 故障排除：05-troubleshooting.md#常见故障模式

#### 部署问题
- 容器化：06-best-practices.md#容器化部署
- 网络配置：05-troubleshooting.md#分布式通信故障
- 负载均衡：06-best-practices.md#高可用架构

### 7.4 📖 扩展阅读

#### 官方文档
- [vLLM官方文档](https://docs.vllm.ai/)
- [CUDA编程指南](https://docs.nvidia.com/cuda/)
- [NVIDIA NCCL文档](https://docs.nvidia.com/deeplearning/nccl/)

#### 性能优化
- [CUDA性能最佳实践](https://developer.nvidia.com/blog/cuda-performance-tips/)
- [大模型推理优化](https://huggingface.co/docs/transformers/performance)
- [FP8量化指南](https://docs.nvidia.com/deeplearning/transformer-engine/)

#### 监控工具
- [Prometheus监控指南](https://prometheus.io/docs/)
- [Grafana可视化](https://grafana.com/docs/)
- [DCGM监控](https://developer.nvidia.com/dcgm)

---

## 快速使用指南

### 新手入门（5分钟）
1. 阅读 `README.md` 了解总体概念
2. 查看 `07-quick-reference.md` #1.1 立即行动清单
3. 使用配置模板快速部署

### 问题排查（10分钟）
1. 查看 `07-quick-reference.md` #2.1 紧急故障快速响应
2. 参考 `05-troubleshooting.md` 详细故障处理
3. 使用诊断命令快速定位问题

### 性能优化（30分钟）
1. 阅读 `02-performance-analysis.md` 理解性能瓶颈
2. 参考 `03-optimization-strategy.md` 制定优化策略
3. 使用 `04-configuration-guide.md` 调整参数配置

### 生产部署（1小时）
1. 学习 `06-best-practices.md` 最佳实践
2. 参考案例研究制定部署方案
3. 建立监控和运维体系

这个快速参考手册为日常使用提供了即查即用的指导，是整个知识体系的实用索引。
