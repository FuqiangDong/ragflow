# DGX Spark 统一内存架构整体配置优化方案

> **适用对象**：NVIDIA DGX Spark（ARM64 + Blackwell GB10，128 GB 统一内存）  
> **目标**：在统一内存架构下，为 RAGFlow + 嵌入模型 + Reranker + MinerU/VLM 提供最优资源配置  
> **最后更新**：2026-06-26

---

## 1. DGX Spark 统一内存架构特点

DGX Spark 采用 NVIDIA Grace CPU + Blackwell GPU 的 SoC 设计，CPU 与 GPU 共享同一物理内存池：

- **总统一内存**：128 GB
- **GPU 显存**：与系统内存共享，无传统 PCIe 拷贝瓶颈
- **CPU**：20 核 Grace Cortex-X4
- **GPU**：Blackwell GB10，支持 `sm_121`

这意味着：

1. GPU 可以访问超过传统独立显存容量的内存。
2. 内存分配需要在 CPU 服务、GPU 推理、系统缓存之间精细平衡。
3. 过度预分配 GPU 内存会挤压 CPU 服务可用内存，反之亦然。

---

## 2. 整体资源分配策略（128 GB 统一内存）

### 2.1 推荐分配表

| 组件 | 推荐内存/显存 | 说明 |
|------|---------------|------|
| 系统保留 + Docker/OS 缓存 | 16 GB | 操作系统、Docker daemon、日志、临时文件 |
| RAGFlow 主容器 | 16 GB | API、任务调度、DeepDoc CPU 推理、Chunking |
| infinity-emb（GPU） | 16 GB | `BAAI/bge-m3` 嵌入 + `BGE-reranker-v2-m3` reranker |
| MinerU / vLLM（GPU） | 32–64 GB | VLM 文档解析，根据并发调整 |
| Elasticsearch | 32 GB | JVM heap（建议 24 GB）+ 文件系统缓存 |
| MySQL + Redis/Valkey + MinIO | 8 GB | 元数据、缓存、对象存储 |
| 预留扩展 | 16–32 GB | 后续模型升级、并发扩容缓冲 |

### 2.2 不同负载下的配置模式

#### 模式 A：轻量解析（嵌入为主）

适合以文本检索为主、PDF 解析量小的场景：

- MinerU / vLLM：16 GB
- infinity-emb：16 GB
- Elasticsearch：24 GB
- RAGFlow 主容器：16 GB
- 其他：16 GB
- 预留：40 GB

#### 模式 B：重 PDF 解析（MinerU 高并发）

适合大量扫描版 PDF、复杂版面的场景：

- MinerU / vLLM：48–64 GB
- infinity-emb：16 GB
- Elasticsearch：24 GB
- RAGFlow 主容器：16 GB
- 其他：8 GB
- 预留：8–20 GB

#### 模式 C：平衡模式（推荐）

适合大多数企业知识库场景：

- MinerU / vLLM：32 GB
- infinity-emb：16 GB
- Elasticsearch：32 GB
- RAGFlow 主容器：16 GB
- 其他：8 GB
- 预留：24 GB

---

## 3. 各组件具体优化配置

### 3.1 RAGFlow 主容器

在 `docker/.env` 中：

```bash
# CPU 模式（ARM64 推荐）
DEVICE=cpu

# 向量引擎：Elasticsearch 在 ARM64 上更稳定
DOC_ENGINE=elasticsearch

# API 代理方案：Python 更稳定，Go 后端可选
API_PROXY_SCHEME=python

# 任务执行器并发，按 CPU 核数调整
MAX_WORKERS=8
```

Docker Compose 资源限制（可选）：

```yaml
services:
  ragflow:
    deploy:
      resources:
        limits:
          cpus: '16'
          memory: 16G
        reservations:
          cpus: '8'
          memory: 8G
```

### 3.2 infinity-emb（GPU Embedding + Reranker）

`docker-compose-dgx-arm64.yml` 中已配置。关键优化：

- 模型自动加载：`BAAI/bge-m3` 和 `BAAI/bge-reranker-v2-m3`
- 批量大小：默认已优化，若显存不足可降低 `BATCH_SIZE`
- 并发：根据查询 QPS 调整 `WORKERS`

显存占用参考：

| 模型 | 显存占用 |
|------|----------|
| bge-m3 | ~4–6 GB |
| bge-reranker-v2-m3 | ~4–6 GB |
| 合计 | ~8–12 GB |

预留 16 GB 足够，并留有余量。

### 3.3 MinerU / vLLM（GPU PDF 解析）

DGX Spark 上**已验证**的部署路径为 `tools/mineru-gpu/` 中的独立 vLLM Server + MinerU API 架构。

#### 独立 vLLM Server（推荐）

在 `tools/mineru-gpu/docker-compose.yml` 的 `mineru-vllm-server` 中调整：

```yaml
command:
  - --host
  - 0.0.0.0
  - --port
  - "30000"
  - --gpu-memory-utilization
  - "0.3"   # 约 36 GB，平衡模式
```

显存利用建议：

| 模式 | `--gpu-memory-utilization` | 约占用显存 | 适用场景 |
|------|---------------------------|-----------|---------|
| 轻量 | 0.20 | ~24 GB | 低并发、简单文档 |
| 平衡 | 0.30 | ~36 GB | 日常办公文档 |
| 重载 | 0.40–0.50 | ~48–60 GB | 高并发、复杂版面/公式 |

并发控制：

```bash
MINERU_API_MAX_CONCURRENT_REQUESTS=3
```

> 仓库根目录的 `Dockerfile.mineru-dgx-arm64`（单容器方案）未在 DGX Spark 实测，不建议作为主要生产路径。

### 3.4 Elasticsearch

在 `docker/.env` 中：

```bash
# JVM heap，建议不超过总内存的 25%，最大 32 GB
ES_JAVA_OPTS=-Xms24g -Xmx24g
```

Docker Compose 资源限制：

```yaml
services:
  es01:
    deploy:
      resources:
        limits:
          memory: 32G
```

优化建议：

- 使用 SSD/NVMe 存储索引数据
- 分片数按知识库数量规划，避免过多小分片
- 定期执行 `forcemerge` 减少段数

### 3.5 MySQL / Redis / MinIO

这些组件内存占用相对稳定，合计 8 GB 足够：

```yaml
services:
  mysql:
    deploy:
      resources:
        limits:
          memory: 4G
  redis:
    deploy:
      resources:
        limits:
          memory: 2G
  minio:
    deploy:
      resources:
        limits:
          memory: 2G
```

---

## 4. 性能调优建议

### 4.1 PDF 解析链路

1. **MinerU 优先处理复杂 PDF**：扫描版、公式多、表格多的文档用 MinerU。
2. **简单 PDF 用 DeepDoc CPU**：纯文本 PDF 用 RAGFlow 内置解析器，节省 GPU。
3. **批量上传控制并发**：避免同时上传大量大文档导致 vLLM OOM。

### 4.2 Embedding 链路

1. **批量请求**：对话检索时尽量批量查询 embedding。
2. **缓存高频查询**：Redis 缓存热门查询结果。
3. **合理设置 top-k**：默认 1024 可根据召回质量调整为 128–512。

### 4.3 Reranker 链路

1. 仅在需要高精度排序时启用 reranker。
2. reranker 输入长度控制在 512 token 以内。
3. 对 reranker 结果截断，避免过长上下文进入 LLM。

### 4.4 LLM 链路

1. 根据任务选择合适模型：简单总结用轻量模型，复杂推理用强模型。
2. 开启流式输出，降低首 token 延迟。
3. 合理设置 `max_tokens`，避免生成过长无意义内容。

---

## 5. 监控建议

### 5.1 系统级监控

```bash
# 统一内存和 GPU 使用
watch -n 2 nvidia-smi

# 容器资源使用
docker stats

# 系统内存
dfree -h
```

### 5.2 服务级监控

| 服务 | 检查命令 |
|------|----------|
| RAGFlow API | `curl http://localhost:9380/` |
| Elasticsearch | `curl http://localhost:9200/_cluster/health` |
| infinity-emb | `curl http://localhost:6900/health` |
| MinerU API | `curl http://localhost:8000/openapi.json` |
| MinerU vLLM | `curl http://localhost:30000/health` |

### 5.3 关键告警指标

- GPU 显存使用率 > 90%
- 统一内存使用率 > 90%
- Elasticsearch JVM heap > 85%
- RAGFlow 任务队列堆积
- MinerU API 5xx 错误率 > 1%

---

## 6. 总结

DGX Spark 的 128 GB 统一内存可以同时支撑 RAGFlow 主服务、GPU embedding、GPU PDF 解析和向量数据库。关键优化原则：

1. **按需分配 GPU 内存**：vLLM 默认会预分配大量显存，需根据实际负载限制。
2. **CPU 与 GPU 服务隔离**：RAGFlow 主容器跑 CPU，推理交给独立 GPU 服务。
3. **避免重复加载模型**：独立 vLLM Server 可被 MinerU 复用。
4. **监控与弹性**：持续监控统一内存和 GPU 使用，按负载动态调整。

推荐大多数企业场景采用**平衡模式**：

- MinerU / vLLM：32 GB（`--gpu-memory-utilization 0.25`）
- infinity-emb：16 GB
- Elasticsearch：32 GB
- RAGFlow + 其他：24 GB
- 预留：24 GB

---

## 7. 参考

- [RAGFlow DGX ARM64 部署指南](./deploy_dgx_arm64.md)
- [RAGFlow + MinerU GPU 部署指南（DGX Spark）](./deploy_mineru_gpu_dgx_arm64.md)
- `tools/mineru-gpu/README.md`
