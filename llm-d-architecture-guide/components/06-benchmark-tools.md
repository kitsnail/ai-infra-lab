# Benchmark Tools 组件详解

## 组件概述

Benchmark Tools是llm-d的自动化性能测试和基准评估工具套件，提供从部署、实验执行到数据收集的完整工作流程。它支持多种负载生成器，具备配置探索和容量规划功能，专为AI推理系统的性能优化设计。

## 核心功能

### 1. 自动化工作流程
- **部署自动化**: 一键部署测试环境和模型服务
- **实验执行**: 自动化基准测试实验运行
- **数据收集**: 系统化性能数据收集和分析
- **结果报告**: 标准化性能报告生成

### 2. 多种负载生成器支持
- **fmperf集成**: 专业的AI推理负载生成器
- **inference-perf集成**: 推理性能专用测试工具
- **guidellm集成**: 大语言模型性能评估工具
- **自定义负载生成器**: 支持自定义测试场景

### 3. 配置探索器
- **参数空间搜索**: 自动探索最优配置参数
- **性能建模**: 基于实验数据构建性能模型
- **优化建议**: 提供具体的性能优化建议
- **对比分析**: 不同配置下的性能对比

### 4. 容量规划工具
- **资源需求预测**: 预测不同负载下的资源需求
- **成本效益分析**: 评估不同部署方案的成本效益
- **扩展性评估**: 评估系统的水平扩展能力
- **SLO验证**: 验证服务水平目标的达成情况

## 技术架构

### 工作流程架构
```yaml
# 基准测试工作流程
workflow:
  phases:
    - setup:           # 环境准备阶段
      - cluster_config
      - model_deployment
      - monitoring_setup
    
    - benchmark:       # 基准测试阶段
      - load_generation
      - data_collection
      - performance_monitoring
    
    - analysis:        # 数据分析阶段
      - data_processing
      - performance_modeling
      - report_generation
    
    - cleanup:         # 清理阶段
      - resource_cleanup
      - data_backup
      - report_delivery
```

### 负载生成器集成
```yaml
# 支持的负载生成器
load_generators:
  fmperf:
    features:
      - multi_model_support
      - advanced_metrics
      - custom_workloads
    config_file: fmperf_config.yaml
    
  inference-perf:
    features:
      - openai_compatible
      - concurrent_testing
      - latency_analysis
    config_file: inference_perf_config.yaml
    
  guidellm:
    features:
      - llm_specific_metrics
      - quality_assessment
      - cost_analysis
    config_file: guidellm_config.yaml
    
  custom:
    features:
      - pluggable_architecture
      - custom_metrics
      - integration_apis
    sdk_language: "python,go,rust"
```

### 数据收集和分析
```yaml
# 数据收集管道
data_pipeline:
  collection:
    sources:
      - prometheus_metrics
      - application_logs
      - system_metrics
      - load_generator_data
    
  processing:
    stages:
      - data_validation
      - normalization
      - aggregation
      - outlier_detection
    
  analysis:
    metrics:
      - throughput_tokens_per_second
      - latency_percentiles
      - resource_utilization
      - cost_per_token
      - error_rates
      
    reports:
      - summary_report
      - detailed_analysis
      - trend_analysis
      - optimization_recommendations
```

## 配置管理

### 基础配置
```yaml
# benchmark-config.yaml
apiVersion: llm-d.io/v1alpha1
kind: BenchmarkConfig
metadata:
  name: llm-d-benchmark
spec:
  # 集群配置
  cluster:
    provider: kubernetes
    namespace: llm-d-benchmark
    monitoring:
      enabled: true
      prometheus_url: "http://prometheus.monitoring.svc.cluster.local:9090"
  
  # 模型配置
  models:
    - name: "DeepSeek-V3.1"
      image: "deepseek-ai/DeepSeek-V3.1"
      replicas: 2
      resources:
        requests:
          nvidia.com/gpu: 8
          memory: "256Gi"
        limits:
          nvidia.com/gpu: 8
          memory: "512Gi"
  
  # 基准测试配置
  benchmark:
    duration: "30m"
    warmup: "5m"
    cooldown: "2m"
    load_generators:
      - type: fmperf
        config:
          concurrent_requests: 10
          request_rate: 5
          input_length: 512
          output_length: 256
      - type: inference-perf
        config:
          threads: 8
          connections: 16
          duration: 1800
```

### 高级配置选项
```yaml
# 高级基准测试配置
advanced_config:
  # 实验设计
  experiment_design:
    type: "factorial_design"  # factorial_design, grid_search, random_search
    factors:
      - name: "batch_size"
        values: [1, 4, 8, 16]
      - name: "max_tokens"
        values: [512, 1024, 2048]
      - name: "temperature"
        values: [0.1, 0.7, 1.0]
    optimization_target: "throughput"
    
  # 性能目标
  performance_targets:
    slos:
      - name: "p95_latency"
        target: "<2000ms"
        weight: 0.4
      - name: "throughput"
        target: ">50 tokens/s"
        weight: 0.3
      - name: "cost_per_token"
        target: "<$0.001"
        weight: 0.3
        
  # 容量规划
  capacity_planning:
    scenarios:
      - name: "peak_load"
        multiplier: 2.0
        duration: "1h"
      - name: "steady_state"
        multiplier: 1.0
        duration: "24h"
      - name: "scale_out"
        target_throughput: 1000
        max_replicas: 10
```

## 部署集成

### 安装和配置
```bash
# 安装llm-d-benchmark
helm repo add llm-d-benchmark https://llm-d-incubation.github.io/llm-d-benchmark/helm
helm install llm-d-benchmark llm-d-benchmark/llm-d-benchmark \
  --namespace llm-d-benchmark \
  --create-namespace \
  --version v0.3.0

# 配置基准测试环境
kubectl apply -f benchmark-config.yaml
```

### 运行基准测试
```bash
# 运行预定义基准测试
kubectl create -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: benchmark-job
spec:
  template:
    spec:
      containers:
      - name: benchmark-runner
        image: ghcr.io/llm-d/llm-d-benchmark:v0.3.0
        command:
        - python
        - -m
        - llm_d_benchmark
        - --config
        - /config/benchmark-config.yaml
        - --output
        - /results
        volumeMounts:
        - name: config
          mountPath: /config
        - name: results
          mountPath: /results
      volumes:
      - name: config
        configMap:
          name: benchmark-config
      - name: results
        persistentVolumeClaim:
          claimName: benchmark-results-pvc
      restartPolicy: Never
EOF
```

### 使用不同负载生成器
```bash
# 使用fmperf进行基准测试
kubectl create job fmperf-benchmark \
  --image=ghcr.io/llm-d/llm-d-benchmark:v0.3.0 \
  -- \
  python -m llm_d_benchmark \
  --load-generator fmperf \
  --config fmperf-config.yaml

# 使用inference-perf进行基准测试
kubectl create job inference-perf-benchmark \
  --image=ghcr.io/llm-d/llm-d-benchmark:v0.3.0 \
  -- \
  python -m llm_d_benchmark \
  --load-generator inference-perf \
  --config inference-perf-config.yaml

# 使用guidellm进行基准测试
kubectl create job guidellm-benchmark \
  --image=ghcr.io/llm-d/llm-d-benchmark:v0.3.0 \
  -- \
  python -m llm_d_benchmark \
  --load-generator guidellm \
  --config guidellm-config.yaml
```

## 使用场景

### 1. 性能评估
```yaml
# 性能评估场景
performance_evaluation:
  objectives:
    - baseline_characterization
    - regression_detection
    - optimization_validation
  
  metrics:
    primary:
      - throughput_tokens_per_second
      - latency_p95
      - gpu_utilization
    secondary:
      - memory_usage
      - error_rate
      - cost_per_token
      
  scenarios:
    - single_model_benchmark
    - multi_model_comparison
    - scaling_analysis
```

### 2. 容量规划
```yaml
# 容量规划场景
capacity_planning:
  use_cases:
    - production_deployment
    - disaster_recovery
    - cost_optimization
    
  analysis:
    - resource_requirements_by_load
    - scaling_recommendations
    - cost_benefit_analysis
    
  outputs:
    - capacity_matrix
    - scaling_curves
    - cost_projections
```

### 3. 配置优化
```yaml
# 配置优化场景
configuration_optimization:
  search_space:
    batch_sizes: [1, 2, 4, 8, 16, 32]
    max_tokens: [256, 512, 1024, 2048, 4096]
    temperatures: [0.1, 0.3, 0.5, 0.7, 1.0]
    
  optimization_methods:
    - grid_search
    - bayesian_optimization
    - genetic_algorithm
    
  constraints:
    max_latency: "2000ms"
    min_throughput: "50 tokens/s"
    max_cost: "$0.001/token"
```

### 4. 回归测试
```yaml
# 回归测试场景
regression_testing:
  baseline_version: "v0.3.0"
  test_version: "v0.4.0"
  
  test_suite:
    - functionality_tests
    - performance_tests
    - stability_tests
    
  pass_criteria:
    performance_regression: "<5%"
    functionality_changes: "compatible"
    stability: ">99.9% uptime"
```

## 监控和可观测性

### 实时监控
```yaml
# 实时监控配置
monitoring:
  metrics:
    - request_rate
    - response_time
    - error_rate
    - resource_utilization
    - throughput
    
  alerts:
    - high_latency: "p95_latency > 5s"
    - high_error_rate: "error_rate > 1%"
    - resource_exhaustion: "memory_usage > 95%"
    
  dashboards:
    - real_time_performance
    - historical_trends
    - comparative_analysis
```

### 数据分析
```bash
# 查看基准测试结果
kubectl get benchmarkresults -o yaml

# 访问详细报告
kubectl port-forward benchmark-dashboard 3000:3000
open http://localhost:3000

# 导出数据进行分析
kubectl cp benchmark-pod:/results ./benchmark-results/
python analyze_results.py benchmark-results/
```

## 性能指标

### 核心指标
```yaml
# 关键性能指标
kpi_metrics:
  throughput:
    - tokens_per_second
    - requests_per_second
    - batches_per_second
    
  latency:
    - time_to_first_token
    - time_per_output_token
    - end_to_end_latency
    - latency_percentiles (p50, p95, p99)
    
  resource_utilization:
    - gpu_utilization_percentage
    - memory_usage_percentage
    - cpu_utilization_percentage
    - network_io_rate
    
  quality:
    - response_coherence
    - content_relevance
    - toxicity_scores
```

### 成本指标
```yaml
# 成本相关指标
cost_metrics:
  compute_cost:
    - gpu_hours
    - cost_per_token
    - cost_per_request
    
  infrastructure_cost:
    - storage_cost
    - network_cost
    - monitoring_cost
    
  operational_cost:
    - maintenance_cost
    - engineering_effort
    - downtime_cost
```

## 故障排查

### 常见问题
```yaml
# 基准测试问题
benchmark_issues:
  # 负载生成器问题
  load_generator_failure:
    symptoms:
      - "No data collected from load generator"
      - "High error rates during test"
    solutions:
      - "Check load generator configuration"
      - "Verify network connectivity"
      - "Increase timeout settings"
      
  # 部署问题
  deployment_failure:
    symptoms:
      - "Pods in CrashLoopBackOff state"
      - "Failed to pull images"
    solutions:
      - "Check image availability and version"
      - "Verify resource quotas"
      - "Review security policies"
      
  # 数据收集问题
  data_collection_failure:
    symptoms:
      - "Missing metrics in results"
      - "Incomplete data sets"
    solutions:
      - "Check Prometheus connectivity"
      - "Verify metric labels"
      - "Review data retention policies"
```

### 调试方法
```bash
# 检查基准测试状态
kubectl get benchmarks -o wide
kubectl describe benchmark benchmark-name

# 查看详细日志
kubectl logs -f benchmark-runner-pod

# 检查资源状态
kubectl top pods -n llm-d-benchmark
kubectl get events -n llm-d-benchmark

# 验证配置
kubectl get configmap benchmark-config -o yaml
```

## 版本演进

### v0.3.0 当前特性
- 支持多种负载生成器集成
- 改进配置探索器功能
- 增强数据分析能力
- 优化成本效益分析
- 改进用户界面和报告

### 从v0.2.0迁移
```yaml
# 配置变更
旧格式: "load_generator: fmperf"
新格式: "load_generators: [{type: fmperf, config: {...}}]"

# API变更
旧版本: POST /api/v1/benchmark
新版本: POST /api/v1alpha1/benchmarks
```

## 最佳实践

### 实验设计
1. **明确目标**: 明确定义性能优化目标和成功标准
2. **控制变量**: 每次实验只改变一个关键参数
3. **重复验证**: 多次运行实验确保结果可靠性
4. **基准对比**: 建立明确的基准线和对比组

### 数据收集
1. **全面监控**: 收集系统、应用和网络各个层面的数据
2. **时间同步**: 确保所有指标的时间戳同步
3. **数据验证**: 验证数据的完整性和准确性
4. **存储策略**: 制定合理的数据存储和备份策略

### 结果分析
1. **统计方法**: 使用合适的统计方法分析数据
2. **可视化**: 通过图表直观展示性能趋势
3. **根因分析**: 深入分析性能瓶颈的根本原因
4. **可重现性**: 确保实验结果的可重现性

### 持续改进
1. **定期评估**: 定期运行性能基准测试
2. **趋势监控**: 监控性能随时间的变化趋势
3. **主动优化**: 基于测试结果进行主动性能优化
4. **知识共享**: 将测试经验和最佳实践文档化

## 与llm-d生态集成

### 与Inference Scheduler集成
- **调度策略评估**: 测试不同调度策略的性能影响
- **缓存效率评估**: 评估KV缓存对不同调度的影响
- **负载均衡测试**: 验证负载均衡算法的效果

### 与Model Service集成
- **部署配置优化**: 测试不同的模型服务部署配置
- **资源分配测试**: 评估不同资源分配策略的效果
- **扩展性验证**: 测试模型服务的水平扩展能力

### 与Inference Simulator集成
- **开发测试**: 在开发阶段使用模拟器进行初步测试
- **CI/CD集成**: 将基准测试集成到持续集成流水线
- **回归测试**: 自动化回归测试检测性能退化
