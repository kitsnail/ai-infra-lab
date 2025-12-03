# 故障排除与监控指南

## 1. 故障分类体系

### 1.1 故障类型矩阵
```
┌─────────────────────────────────────────────────────────────┐
│                    故障类型分类                           │
├─────────────────────────────────────────────────────────────┤
│ 🔴 严重故障: 服务不可用，完全失效                           │
│ 🟡 性能故障: 服务可用但性能下降                             │
│ 🟠 配置故障: 参数错误导致异常                               │
│ 🔵 资源故障: 硬件资源不足或异常                             │
│ 🟢 预警故障: 潜在问题，需要关注                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 故障影响范围
```
┌─────────────────┬─────────────────┬─────────────────┐
│ 故障级别       │ 影响范围        │ 恢复时间        │
├─────────────────┼─────────────────┼─────────────────┤
│ P0 - 紧急      │ 全部用户        │ <5分钟         │
│ P1 - 高        │ 主要用户        │ <15分钟        │
│ P2 - 中        │ 部分用户        │ <1小时         │
│ P3 - 低        │ 单个用户        │ <4小时         │
│ P4 - 预警      │ 无直接影响      │ <24小时        │
└─────────────────┴─────────────────┴─────────────────┘
```

## 2. 🔴 常见故障模式

### 2.1 OOM (Out of Memory) 故障

#### 2.1.1 故障现象
```
错误信息:
- "CUDA out of memory"
- "RuntimeError: CUDA error: out of memory"
- "torch.cuda.OutOfMemoryError"

服务状态:
- 进程崩溃或重启
- 请求返回500错误
- GPU内存使用率100%
```

#### 2.1.2 根因分析
```
主要原因:
1. 模型权重 + KV-Cache + 激活内存 > GPU可用内存
2. 并发请求过多，KV-Cache累积
3. 内存碎片化严重
4. GPU内存利用率设置过高 (>95%)

计算验证:
total_memory = model_weights + kv_cache + activations + system_reserved
```

#### 2.1.3 解决方案
```
紧急恢复:
1. 降低gpu-memory-utilization (0.95 → 0.85)
2. 减少max-num-seqs (128 → 64)
3. 重启服务清理内存

长期优化:
1. 启用FP8 KV-Cache
2. 优化批处理大小
3. 实施内存监控预警
4. 考虑模型分布式部署

配置调整:
--gpu-memory-utilization 0.85
--max-num-seqs 64
--kv-cache-dtype "fp8"
```

### 2.2 性能骤降故障

#### 2.2.1 故障现象
```
性能指标:
- 吞吐量下降 >50%
- P50延迟增加 >100%
- 请求错误率 >5%

服务状态:
- 响应超时增多
- 用户体验明显下降
- 监控告警触发
```

#### 2.2.2 根因分析
```
常见原因:
1. 启用了--enforce-eager (性能杀手)
2. GPU硬件异常 (降频、过热)
3. 网络通信问题 (分布式场景)
4. 模型文件损坏或异常
5. 系统资源竞争

诊断步骤:
1. 检查配置中是否包含enforce-eager
2. 查看GPU状态和温度
3. 测试网络延迟和带宽
4. 验证模型文件完整性
5. 检查系统资源使用
```

#### 2.2.3 解决方案
```
立即措施:
1. 移除--enforce-eager参数
2. 重启GPU服务
3. 检查硬件状态
4. 验证配置正确性

预防措施:
1. 配置模板化，避免错误参数
2. 建立性能基线监控
3. 实施配置变更审查
4. 定期硬件维护检查
```

### 2.3 服务启动失败

#### 2.3.1 故障现象
```
错误类型:
- 模型加载失败
- GPU初始化错误
- 端口占用问题
- 权限不足错误

典型错误信息:
- "No such file or directory: model-path"
- "CUDA initialization error"
- "Address already in use"
- "Permission denied"
```

#### 2.3.2 解决方案
```
模型加载问题:
1. 检查模型路径和权限
   ls -la /model-cache/Qwen/
2. 验证模型文件完整性
   md5sum model-weights.bin
3. 检查磁盘空间
   df -h /model-cache/

GPU初始化问题:
1. 检查GPU状态
   nvidia-smi
2. 验证CUDA驱动
   nvcc --version
3. 检查GPU内存
   nvidia-smi --query-gpu=memory.total,memory.free

网络端口问题:
1. 检查端口占用
   netstat -tlnp | grep 8200
2. 修改端口配置
   --port 8201
3. 检查防火墙设置
   iptables -L

权限问题:
1. 检查用户权限
   id
   groups
2. 检查文件权限
   chmod +x scripts/
3. 检查Docker权限
   docker run --gpus all
```

### 2.4 分布式通信故障

#### 2.4.1 故障现象
```
通信错误:
- NCCL初始化失败
- 节点间通信超时
- 网络带宽异常
- 集群状态不一致

错误信息:
- "NCCL initialization failed"
- "Connection timeout"
- "Network unreachable"
- "Backend initialization error"
```

#### 2.4.2 诊断与解决
```
网络层检查:
1. 测试节点连通性
   ping node2
   traceroute node2
2. 检查网络带宽
   iperf3 -c node2 -t 30
3. 验证端口开放
   telnet node2 12345

NCCL配置检查:
1. 检查NCCL环境变量
   env | grep NCCL
2. 验证网络接口
   NCCL_SOCKET_IFNAME=eth0
3. 测试InfiniBand状态
   ibstat

分布式配置检查:
1. 验证Ray集群状态
   ray status
2. 检查节点注册
   ray nodes
3. 查看资源分配
   ray resources

解决方案:
export NCCL_DEBUG=INFO
export NCCL_SOCKET_IFNAME=eth0
export NCCL_IB_DISABLE=0
export GLOO_SOCKET_IFNAME=eth0
```

## 3. 🟡 性能问题诊断

### 3.1 吞吐量下降诊断

#### 3.1.1 性能指标分析
```
正常基准 (基于测试结果):
- 吞吐量: >1800 tok/s
- P50延迟: <2s
- P95延迟: <8s
- GPU利用率: >85%

异常阈值:
- 吞吐量下降 >15%: 告警
- 延迟增加 >30%: 警告
- GPU利用率 <70%: 异常
- 错误率 >1%: 严重
```

#### 3.1.2 诊断流程
```
步骤1: 配置检查
□ 是否包含--enforce-eager
□ GPU内存利用率设置
□ 并发参数配置
□ 数据类型选择

步骤2: 资源检查
□ GPU状态 (nvidia-smi)
□ 内存使用情况
□ CPU利用率
□ 网络I/O

步骤3: 应用检查
□ 服务进程状态
□ 请求队列长度
□ 错误日志分析
□ 性能指标趋势

步骤4: 环境检查
□ 系统负载情况
□ 其他进程干扰
□ 网络延迟测试
□ 存储I/O性能
```

#### 3.1.3 常见性能杀手
```
配置层面:
1. --enforce-eager (-153%性能)
2. --gpu-memory-utilization 1.0 (OOM风险)
3. --dtype float32 (性能低50%)
4. --max-num-seqs 过小 (并发受限)

硬件层面:
1. GPU降频 (温度过高)
2. 网络带宽不足
3. 存储I/O瓶颈
4. 内存碎片化

应用层面:
1. 批处理效率低
2. KV-Cache命中率低
3. 调度策略不当
4. 负载不均衡
```

### 3.2 延迟问题诊断

#### 3.2.1 延迟组成分析
```
总延迟 = TTFT + (输出token数 × TPOT)

其中:
- TTFT (Time to First Token): 首token延迟
- TPOT (Time per Output Token): 输出token间隔延迟

正常范围:
- TTFT: <2s (P50)
- TPOT: <50ms
- P95: <8s
```

#### 3.2.2 延迟优化检查点
```
TTFT优化:
□ 启用Chunked Prefill
□ 优化批处理大小
□ 减少模型加载延迟
□ 网络延迟优化

TPOT优化:
□ 提升GPU利用率
□ 优化Kernel执行效率
□ 减少同步开销
□ 启用CUDA Graph

尾部延迟优化:
□ 调整并发参数
□ 优化调度策略
□ 避免资源竞争
□ 实施负载均衡
```

## 4. 🔵 监控体系建设

### 4.1 核心监控指标

#### 4.1.1 业务指标
```
吞吐量指标:
- Input Token Throughput (tok/s)
- Output Token Throughput (tok/s)
- Total Token Throughput (tok/s)
- Request Throughput (req/s)

延迟指标:
- TTFT (Time to First Token)
- TPOT (Time per Output Token)
- ITL (Inter-Token Latency)
- Overall Latency (P50/P95/P99)

质量指标:
- Request Success Rate (%)
- Error Rate (%)
- Timeout Rate (%)
- Output Quality Score
```

#### 4.1.2 系统指标
```
GPU指标:
- GPU Utilization (%)
- GPU Memory Utilization (%)
- GPU Temperature (°C)
- GPU Power Usage (W)

内存指标:
- KV-Cache Memory (GB)
- Model Weights Memory (GB)
- Activations Memory (GB)
- Memory Fragmentation Rate

网络指标:
- Network Throughput (Gbps)
- Network Latency (ms)
- Packet Loss Rate (%)
- Connection Count

系统指标:
- CPU Utilization (%)
- System Memory Usage (%)
- Disk I/O (IOPS)
- Process Status
```

### 4.2 告警策略

#### 4.2.1 告警级别定义
```
Critical (紧急):
- 服务不可用 (HTTP 5xx > 5%)
- GPU内存OOM (>0次/小时)
- 吞吐量下降 >50%
- P95延迟 >15s

Warning (警告):
- 吞吐量下降 >20%
- P50延迟 >4s
- GPU利用率 <70%
- 错误率 >3%

Info (信息):
- 吞吐量下降 >10%
- P50延迟 >3s
- GPU利用率 <80%
- 错误率 >1%
```

#### 4.2.2 告警响应流程
```
P0 告警 (Critical):
1. 自动通知: 立即电话+短信
2. 响应时间: <5分钟
3. 处理流程: 紧急故障处理
4. 升级机制: 15分钟无响应自动升级

P1 告警 (Warning):
1. 自动通知: 企业微信+邮件
2. 响应时间: <15分钟
3. 处理流程: 标准故障处理
4. 升级机制: 1小时无响应升级

P2 告警 (Info):
1. 自动通知: 企业微信
2. 响应时间: <1小时
3. 处理流程: 日常维护处理
4. 升级机制: 4小时无响应升级
```

### 4.3 监控工具链

#### 4.3.1 基础监控
```
系统监控:
- Prometheus + Grafana
- Node Exporter
- GPU Exporter (DCGM)
- Blackbox Exporter

应用监控:
- 自定义metrics暴露
- 日志收集 (ELK Stack)
- 链路追踪 (Jaeger)
- 错误追踪 (Sentry)
```

#### 4.3.2 业务监控
```
性能监控:
- 实时QPS/TPS监控
- 延迟分布监控
- 吞吐量趋势监控
- 错误率监控

可用性监控:
- 健康检查监控
- 端点可用性监控
- 服务响应时间监控
- 业务指标监控
```

### 4.4 监控Dashboard

#### 4.4.1 核心Dashboard
```
1. 实时性能Dashboard
   - 实时吞吐量曲线
   - 延迟分布热力图
   - GPU资源使用率
   - 错误率趋势

2. 资源使用Dashboard
   - GPU利用率历史
   - 内存使用趋势
   - 网络流量监控
   - 系统负载监控

3. 业务质量Dashboard
   - 请求成功率
   - 用户满意度指标
   - SLA达成率
   - 故障统计
```

#### 4.4.2 告警Dashboard
```
1. 告警状态总览
   - 活跃告警数量
   - 告警级别分布
   - 告警处理进度
   - MTTR统计

2. 告警趋势分析
   - 告警频率趋势
   - 故障模式分析
   - 重复告警识别
   - 根因分析结果
```

## 5. 🔧 故障处理工具

### 5.1 诊断脚本

#### 5.1.1 系统健康检查
```bash
#!/bin/bash
# health_check.sh - 系统健康检查脚本

echo "=== GPU状态检查 ==="
nvidia-smi --query-gpu=index,name,utilization.gpu,utilization.memory,memory.total,memory.used,temperature.gpu --format=csv

echo "=== 内存使用检查 ==="
free -h

echo "=== 磁盘空间检查 ==="
df -h /model-cache

echo "=== 网络连通性检查 ==="
ping -c 3 ray-head

echo "=== 服务进程检查 ==="
ps aux | grep vllm

echo "=== 端口占用检查 ==="
netstat -tlnp | grep 8200
```

#### 5.1.2 性能基准测试
```bash
#!/bin/bash
# benchmark.sh - 性能基准测试脚本

echo "=== 运行性能基准测试 ==="
curl -X POST http://localhost:8200/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3-VL-235B-A22B-Instruct",
    "prompt": "你好，请介绍一下人工智能。",
    "max_tokens": 100,
    "temperature": 0.7
  }' --time

echo "=== 并发测试 ==="
for i in {1..10}; do
  curl -X POST http://localhost:8200/v1/completions \
    -H "Content-Type: application/json" \
    -d '{"model": "Qwen3-VL-235B-A22B-Instruct", "prompt": "测试", "max_tokens": 50}' &
done
wait
```

### 5.2 自动恢复机制

#### 5.2.1 服务自愈脚本
```bash
#!/bin/bash
# auto_recovery.sh - 服务自动恢复脚本

SERVICE_NAME="vllm-server"
LOG_FILE="/var/log/vllm/recovery.log"

check_service_health() {
    local health_check_url="http://localhost:8200/health"
    local response=$(curl -s -o /dev/null -w "%{http_code}" "$health_check_url")
    return $([ "$response" = "200" ])
}

restart_service() {
    echo "$(date): 服务异常，正在重启..." >> $LOG_FILE
    
    # 停止服务
    pkill -f "vllm.entrypoints"
    sleep 5
    
    # 清理GPU内存
    nvidia-smi --gpu-reset
    
    # 启动服务
    cd /opt/vllm
    nohup ./start_vllm.sh >> /var/log/vllm/server.log 2>&1 &
    
    sleep 30
    echo "$(date): 服务重启完成" >> $LOG_FILE
}

# 主循环
while true; do
    if ! check_service_health; then
        restart_service
    fi
    sleep 60
done
```

#### 5.2.2 资源监控脚本
```bash
#!/bin/bash
# resource_monitor.sh - 资源监控脚本

ALERT_EMAIL="admin@example.com"
GPU_MEM_THRESHOLD=90
GPU_UTIL_THRESHOLD=80

check_gpu_resources() {
    local gpu_util=$(nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits)
    local gpu_mem=$(nvidia-smi --query-gpu=utilization.memory --format=csv,noheader,nounits)
    
    if [ "$gpu_util" -lt "$GPU_UTIL_THRESHOLD" ]; then
        echo "警告: GPU利用率过低 $gpu_util% < $GPU_UTIL_THRESHOLD%"
    fi
    
    if [ "$gpu_mem" -gt "$GPU_MEM_THRESHOLD" ]; then
        echo "警告: GPU内存使用率过高 $gpu_mem% > $GPU_MEM_THRESHOLD%"
    fi
}

# 定期检查
while true; do
    check_gpu_resources
    sleep 300  # 每5分钟检查一次
done
```

### 5.3 调试工具集

#### 5.3.1 性能分析工具
```bash
# performance_profiler.py - Python性能分析工具
import time
import requests
import statistics
from concurrent.futures import ThreadPoolExecutor

class VLLMProfiler:
    def __init__(self, base_url="http://localhost:8200"):
        self.base_url = base_url
        
    def single_request_latency(self, prompt, max_tokens=100):
        """单请求延迟测试"""
        start_time = time.time()
        
        response = requests.post(f"{self.base_url}/v1/completions", 
            json={
                "model": "Qwen3-VL-235B-A22B-Instruct",
                "prompt": prompt,
                "max_tokens": max_tokens
            })
            
        end_time = time.time()
        return end_time - start_time, response.json()
    
    def concurrent_test(self, prompt, num_requests=10):
        """并发请求测试"""
        latencies = []
        
        with ThreadPoolExecutor(max_workers=num_requests) as executor:
            futures = [
                executor.submit(self.single_request_latency, prompt) 
                for _ in range(num_requests)
            ]
            
            for future in futures:
                latency, _ = future.result()
                latencies.append(latency)
        
        return {
            "avg_latency": statistics.mean(latencies),
            "p50": statistics.median(latencies),
            "p95": sorted(latencies)[int(len(latencies) * 0.95)],
            "p99": sorted(latencies)[int(len(latencies) * 0.99)],
            "throughput": num_requests / sum(latencies)
        }

if __name__ == "__main__":
    profiler = VLLMProfiler()
    
    # 单请求测试
    latency, response = profiler.single_request_latency("测试文本", 50)
    print(f"单请求延迟: {latency:.2f}s")
    
    # 并发测试
    results = profiler.concurrent_test("并发测试文本", 20)
    print(f"并发测试结果: {results}")
```

## 6. 🔄 预防性维护策略

### 6.1 定期检查清单

#### 6.1.1 每日检查项
```
□ 服务运行状态检查
□ GPU状态和温度监控
□ 基础性能指标验证
□ 错误日志分析
□ 磁盘空间检查
□ 网络连通性测试
□ 备份完整性验证
```

#### 6.1.2 每周检查项
```
□ 配置参数审查
□ 性能趋势分析
□ 容量规划评估
□ 安全更新检查
□ 监控告警测试
□ 文档更新维护
□ 团队知识分享
```

#### 6.1.3 每月检查项
```
□ 系统性能基准测试
□ 硬件维护检查
□ 灾难恢复演练
□ 架构优化评估
□ 成本效益分析
□ 技术债务清理
□ 技能培训计划
```

### 6.2 容量规划

#### 6.2.1 性能容量模型
```
吞吐量模型:
当前容量: 2000 tok/s
峰值需求: 1500 tok/s
安全边际: 25%
扩展阈值: >1600 tok/s (80%利用率)

延迟模型:
P50目标: <2s
P95目标: <8s
当前P50: 1.4s
当前P95: 17s (排队效应)

资源模型:
GPU利用率: 当前85%，目标90%
内存使用: 当前90%，目标85%
网络带宽: 当前40%，目标60%
```

#### 6.2.2 扩展策略
```
水平扩展 (Scale-out):
- 增加GPU服务器数量
- 实施负载均衡
- 分布式部署架构

垂直扩展 (Scale-up):
- 升级GPU硬件
- 增加系统内存
- 优化存储性能

混合扩展:
- 结合垂直和水平扩展
- 根据负载特性选择
- 成本效益优化
```

### 6.3 变更管理

#### 6.3.1 变更流程
```
变更申请:
1. 变更需求描述
2. 影响范围评估
3. 风险分析报告
4. 回滚计划制定
5. 审批流程

变更实施:
1. 维护窗口安排
2. 备份当前状态
3. 实施变更
4. 验证功能
5. 监控指标

变更验证:
1. 性能基准对比
2. 功能正确性验证
3. 稳定性测试
4. 用户体验评估
5. 文档更新
```

#### 6.3.2 回滚机制
```
快速回滚:
1. 一键回滚脚本
2. 配置版本管理
3. 自动化部署工具
4. 监控告警触发
5. 应急响应流程

完整回滚:
1. 服务停止
2. 环境清理
3. 旧版本部署
4. 数据一致性检查
5. 服务恢复验证
```

## 7. 📊 知识库建设

### 7.1 故障案例库

#### 7.1.1 故障记录模板
```
故障基本信息:
- 故障时间: YYYY-MM-DD HH:MM:SS
- 故障级别: P0/P1/P2/P3
- 影响范围: 用户数量/业务影响
- 故障时长: MTTR

故障描述:
- 故障现象详细描述
- 错误信息和日志
- 监控指标变化
- 用户反馈情况

根因分析:
- 直接原因分析
- 间接原因追溯
- 根本原因定位
- 预防措施建议

处理过程:
- 发现时间线
- 处理步骤记录
- 解决方案选择
- 效果验证结果

经验总结:
- 处理经验提炼
- 工具方法改进
- 流程优化建议
- 知识传承内容
```

#### 7.1.2 故障模式库
```
性能模式:
- 吞吐量下降模式
- 延迟突增模式
- 资源利用率异常模式
- 错误率上升模式

故障模式:
- OOM故障模式
- 网络通信故障模式
- 硬件故障模式
- 配置错误模式

恢复模式:
- 服务重启恢复模式
- 配置回滚恢复模式
- 资源扩容恢复模式
- 降级服务恢复模式
```

### 7.2 最佳实践库

#### 7.2.1 配置最佳实践
```
性能优化:
1. 始终避免--enforce-eager参数
2. 合理设置GPU内存利用率(0.85-0.90)
3. 选择合适的数据类型(bfloat16+FP8)
4. 优化并发参数避免资源竞争

稳定性保障:
1. 配置变更必须经过测试验证
2. 重要配置需要有版本管理
3. 实施监控告警和自动恢复
4. 定期进行故障演练

运维管理:
1. 建立完善的监控体系
2. 制定标准化的运维流程
3. 维护详细的文档和知识库
4. 持续进行团队培训
```

#### 7.2.2 故障处理最佳实践
```
快速响应:
1. 建立分级告警机制
2. 准备应急预案和脚本
3. 确保7x24小时值班
4. 建立升级响应流程

问题定位:
1. 系统化诊断流程
2. 完整的日志收集
3. 分层问题排查方法
4. 经验知识库支持

问题解决:
1. 优先恢复服务可用性
2. 根本原因彻底解决
3. 预防措施有效落地
4. 经验总结和分享
```

---

## 运维工程师要点

1. **预防为主**: 通过监控和定期检查预防故障
2. **快速响应**: 建立标准化故障处理流程
3. **知识积累**: 维护故障案例库和最佳实践
4. **持续改进**: 基于故障经验优化系统架构
5. **团队协作**: 建立有效的沟通和协作机制

这个故障排除指南提供了完整的故障处理体系，确保系统能够稳定高效运行。
