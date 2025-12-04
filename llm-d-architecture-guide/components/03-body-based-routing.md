# Body-Based Router (BBR) 组件详解

## 核心功能

Body-Based Router是llm-d的多模型智能路由组件，通过解析请求体中的`model`字段实现动态路由，作为ExtProc服务器运行，与Kubernetes Gateway集成。

**关键优势**: 多模型无需路径分离，统一API入口

## 核心特性

| 功能 | 描述 | 支持模式 |
|------|------|----------|
| **请求体解析** | 从OpenAI兼容API提取model字段 | streaming/non-streaming |
| **头部注入** | 自动设置`X-Gateway-Model-Name` | JSON解析 |
| **动态路由** | 基于模型名称路由到InferencePool | 统一路径`/` |
| **多模型支持** | 每个模型独立InferencePool | 资源分离，独立扩展 |

## ExtProc集成

### 核心处理流程
```
客户端请求 → Envoy Gateway → BBR ExtProc → HTTPRoute → InferencePool → vLLM实例
                   ↓              ↓                ↓
              解析model字段    设置X-Gateway-Model-Name  路由匹配
```

### 链式配置 (v1.3.1验证)
```yaml
# EnvoyFilter链式处理 - BBR第一，EPP第二
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: body-based-router
  namespace: llm
spec:
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        listener:
          filterChain:
            filter:
              name: envoy.filters.network.http_connection_manager
      patch:
        operation: INSERT_FIRST  # BBR 第一
        value:
          name: envoy.filters.http.ext_proc
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.ext_proc.v3.ExternalProcessor
            allow_mode_override: true
            failure_mode_allow: false
            grpc_service:
              envoy_grpc:
                cluster_name: outbound|9004||body-based-router.llm.svc.cluster.local
              timeout: 1s
            processing_mode:
              request_body_mode: FULL_DUPLEX_STREAMED
              request_header_mode: SEND
              request_trailer_mode: SEND
              response_body_mode: NONE
              response_header_mode: SKIP
              response_trailer_mode: SKIP
    - applyTo: HTTP_FILTER
      patch:
        operation: INSERT_AFTER   # EPP 第二
        value:
          name: envoy.filters.http.ext_proc
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.ext_proc.v3.ExternalProcessor
            allow_mode_override: true
            failure_mode_allow: false
            grpc_service:
              envoy_grpc:
                cluster_name: outbound|9002||gaie-qwen3-235b-vl-epp.llm.svc.cluster.local
              timeout: 1s
            processing_mode:
              request_body_mode: BUFFERED
              request_header_mode: SEND
              request_trailer_mode: SEND
              response_body_mode: NONE
              response_header_mode: SKIP
              response_trailer_mode: SKIP
```

## 部署配置

### Helmwave部署
```bash
# 安装Body-Based Router
helm install body-based-router \
  --set provider.name=istio \
  --version v0.5.1 \
  oci://registry.k8s.io/gateway-api-inference-extension/charts/body-based-routing \
  -n llm

# 应用链式配置
kubectl apply -f envoyfilter-v2.yaml -n llm
```

### Kgateway原生支持
```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: AgentgatewayPolicy
metadata:
  name: bbr
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: llm-d-infra-inference-gateway
  traffic:
    phase: PreRouting
  transformation:
    request:
      set:
      - name: X-Gateway-Model-Name
        value: 'json(request.body).model'
```

### HTTPRoute配置
```yaml
# 模型A路由
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: model-a-route
spec:
  parentRefs:
  - name: llm-d-infra-inference-gateway
  rules:
  - backendRefs:
    - group: inference.networking.x-k8s.io
      kind: InferencePool
      name: pool-a
    matches:
    - path:
        type: PathPrefix
        value: /
      headers:
      - type: Exact
        name: X-Gateway-Model-Name
        value: "Model-A-Name"
```

## 多模型部署流程

```bash
# 1. 部署模型A服务
helm install ms-model-a \
  --set model.name=Model-A-Name \
  --set replicas=2 \
  oci://ghcr.io/llm-d-incubation/charts/llm-d-modelservice \
  -n llm

# 2. 部署模型A池
helm install pool-a \
  --set modelServers.matchLabels.app=ms-model-a \
  --set provider.name=istio \
  oci://registry.k8s.io/gateway-api-inference-extension/charts/inferencepool \
  -n llm

# 3. 创建模型A路由
kubectl apply -f model-a-httproute.yaml

# 4. 重复步骤1-3部署模型B...
```

## 性能优化

### 处理模式选择
```yaml
# Streaming模式 - 适合长请求流式处理
processing_mode:
  request_body_mode: FULL_DUPLEX_STREAMED
  # 优点: 流式处理
  # 缺点: 资源消耗较大

# Buffer模式 - 资源消耗小，延迟低
processing_mode:
  request_body_mode: BUFFERED
  # 优点: 资源消耗小，延迟低
  # 缺点: 不适合极大请求体
```

### 超时和错误处理
```yaml
grpc_service:
  envoy_grpc:
    cluster_name: outbound|9004||body-based-router
    timeout: 1s
allow_mode_override: true
failure_mode_allow: false  # 严格模式，失败不继续
```

## 故障排查

### 常见问题
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **请求未被BBR处理** | EnvoyFilter未正确配置或链式顺序错误 | 检查EnvoyFilter patch顺序和cluster配置 |
| **路由匹配失败** | HTTPRoute headers匹配规则错误 | 验证X-Gateway-Model-Name头部和模型名称 |
| **ExtProc连接失败** | Service名称或端口配置错误 | 检查Service discovery和端口映射 |

### 调试命令
```bash
# 检查BBR处理日志
kubectl logs -f body-based-router-xxx | grep -E "(Processed|set_headers)"

# 检查Envoy配置
kubectl exec -it <istio-gateway-pod> -- curl localhost:15000/config_dump

# 验证HTTPRoute
kubectl get httproute -o yaml
kubectl describe httproute <route-name>

# 测试BBR服务
kubectl port-forward body-based-router-xxx 9004:9004
```

## 监控指标

### BBR专用指标
```yaml
metrics:
  - ext_proc_requests_total
  - ext_proc_request_duration_seconds
  - ext_proc_parsing_errors_total
  - ext_proc_header_injection_total
  - processing_time_histogram
  - body_size_bytes
  - model_match_rate
```

## 版本信息

- **当前版本**: v1.0.0
- **核心特性**:
  - 统一路径多模型路由
  - 请求体解析和头部注入
  - 与EPP链式配置
  - streaming/buffer双模式支持

## 最佳实践

### 生产部署
1. **高可用**: 至少2个BBR副本
2. **监控**: 集成Prometheus监控ExtProc指标
3. **日志**: 启用结构化日志和请求追踪
4. **测试**: 部署前进行多模型路由测试

### 性能调优
1. **处理模式**: 根据请求特征选择streaming或buffer模式
2. **超时配置**: 合理设置gRPC超时时间
3. **资源分配**: 根据请求量调整CPU/内存配额
4. **链式优化**: 确保BBR在链首，EPP在链尾
