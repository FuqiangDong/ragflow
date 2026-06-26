# RAGFlow + MinerU GPU 部署指南（DGX Spark / ARM64 / Blackwell）

> **版本**：RAGFlow v0.26.1 + MinerU ≥ 3.4.0  
> **适用对象**：NVIDIA DGX Spark（ARM64 + Blackwell GB10）以及兼容的 ARM64 GPU 服务器  
> **目标**：在 RAGFlow 中接入 GPU 加速的 MinerU PDF 解析服务  
> **部署入口**：`tools/mineru-gpu/`（本指南以此为准）  
> **最后更新**：2026-06-26

---

## 1. 方案概述

MinerU 是 RAGFlow 支持的 PDF 版面解析引擎之一。在 DGX Spark 上，利用 GB10 的统一内存可以显著加速复杂版面的识别、公式与表格提取。

本部署采用 **独立 vLLM 后端 + MinerU HTTP API** 架构：

```text
┌────────────────────────────────────────────────────────────┐
│                       docker_ragflow                       │
│  ┌─────────────────────┐      ┌─────────────────────┐      │
│  │ mineru-vllm-server  │◄─────│     mineru-api      │      │
│  │      :30000         │      │      :8000          │      │
│  │   (GPU VLM 推理)     │      │   (HTTP API 服务)    │      │
│  └─────────────────────┘      └──────────┬──────────┘      │
│                                          │                  │
│                              ┌───────────▼───────────┐      │
│                              │        ragflow        │      │
│                              │  (MinerU 模型提供者)   │      │
│                              └───────────────────────┘      │
└────────────────────────────────────────────────────────────┘
```

**为什么采用这种架构？**

- **避免 vLLM 双实例冲突**：如果 MinerU API 自己再启动一个 vLLM，会与独立 vLLM Server 重复占用显存，在 128GB 统一内存下也容易 OOM。
- **显存可控**：可以单独调节 `mineru-vllm-server` 的 `--gpu-memory-utilization`。
- **已被验证**：该方案在 DGX Spark 上已实际跑通，配套代码修复在 `deepdoc/parser/mineru_parser.py` 中。

> 仓库根目录的 `Dockerfile.mineru-dgx-arm64` 和 `docker/docker-compose-mineru-dgx-arm64.yml` 是早期单容器方案，未在 DGX Spark 实测，**不建议作为主要生产路径**。

---

## 2. 前置条件

### 2.1 硬件

- NVIDIA DGX Spark 或兼容 ARM64 服务器
- Blackwell GB10 GPU（或兼容 CUDA 的 ARM64 GPU）
- 推荐统一内存 ≥ 128 GB
- SSD/NVMe 存储，模型与数据盘 ≥ 2 TB

### 2.2 软件

- 已完成 `docs/administrator/deploy_dgx_arm64.md` 中的 RAGFlow 主部署
- `docker_ragflow` 网络已存在（由 `docker-compose-dgx-arm64.yml` 创建）
- NVIDIA Container Toolkit ≥ 2.17
- Docker Compose ≥ 2.20

### 2.3 确认 RAGFlow 主服务已启动

```bash
cd docker
docker compose -f docker-compose-dgx-arm64.yml ps
```

应看到 `ragflow`、`es01`、`mysql`、`minio`、`redis`、`infinity-emb` 等容器处于 `Up` 状态，且存在网络：

```bash
docker network ls | grep docker_ragflow
```

---

## 3. 部署步骤

所有操作默认在 `tools/mineru-gpu/` 目录下进行。

### 3.1 进入目录

```bash
cd /home/d0ngfq/project/ragflow/tools/mineru-gpu
```

### 3.2 构建镜像

```bash
# 国内网络建议先配置 Docker 镜像加速
docker build -t mineru:latest -f Dockerfile .
```

**DGX Spark / Blackwell 特殊处理**：

GB10（`sm_121`）需要 vLLM nightly 镜像才包含对应 CUDA 架构支持。编辑 `Dockerfile`，将基础镜像改为：

```dockerfile
FROM vllm/vllm-openai:nightly
```

> 注意：ModelScope 是模型下载源，不是 Docker 镜像仓库，无法直接用来拉取 vLLM 基础镜像。

### 3.3 准备模型目录

模型持久化到宿主机目录 `/home/d0ngfq/models/mineru/models/`：

```bash
mkdir -p /home/d0ngfq/models/mineru/models
```

如果已提前下载模型，目录结构应为：

```text
/home/d0ngfq/models/mineru/models/
├── PDF-Extract-Kit-1.0/
└── MinerU2.5-2509-1.2B/
```

如果目录为空，可用容器先下载模型（ModelScope 源）：

```bash
docker run --rm \
  -v /home/d0ngfq/models/mineru/models:/models \
  mineru:latest \
  python3 -c "from modelscope import snapshot_download; snapshot_download('OpenDataLab/MinerU2.5-2509-1.2B', local_dir='/models/MinerU2.5-2509-1.2B'); snapshot_download('OpenDataLab/PDF-Extract-Kit-1.0', local_dir='/models/PDF-Extract-Kit-1.0')"
```

### 3.4 检查配置文件

`mineru.json` 中模型路径默认如下，需与实际目录名称一致：

```json
{
  "models-dir": {
    "pipeline": "/vllm-workspace/mineru_models/PDF-Extract-Kit-1.0",
    "vlm": "/vllm-workspace/mineru_models/MinerU2.5-2509-1.2B"
  }
}
```

### 3.5 配置环境变量

```bash
cp .env.example .env
```

默认内容：

```bash
MINERU_MODEL_HOST_DIR=/home/d0ngfq/models/mineru/models
MINERU_VLLM_PORT=30000
MINERU_API_PORT=8000
```

按需修改端口或模型路径。

### 3.6 启动服务

```bash
docker compose up -d
```

查看日志：

```bash
# 全部日志
docker compose logs -f

# 单独查看
docker logs -f mineru-vllm-server
docker logs -f mineru-api
```

> 首次启动 `mineru-vllm-server` 时，vLLM 会编译 CUDA graph 并加载模型，可能需要 2–5 分钟才能通过健康检查，请耐心等待。

---

## 4. 验证服务

### 4.1 vLLM 后端

```bash
curl http://localhost:30000/health
```

### 4.2 MinerU API

```bash
curl http://localhost:8000/health
curl http://localhost:8000/openapi.json
```

### 4.3 GPU 是否被使用

```bash
docker exec mineru-vllm-server nvidia-smi
```

### 4.4 解析测试

```bash
curl -X POST "http://localhost:8000/file_parse" \
  -H "Content-Type: multipart/form-data" \
  -F "files=@/path/to/your.pdf" \
  -F "backend=vlm-http-client" \
  -F "parse_method=auto"
```

---

## 5. 在 RAGFlow 中配置 MinerU

### 5.1 添加模型

进入 **User Settings -> Model Provider -> MinerU**：

| 字段 | 推荐值 |
|------|--------|
| Instance Name | `mineru-local` |
| Model Name | `mineru` |
| MinerU API Server | `http://mineru-api:8000` |
| MinerU Backend | `vlm-http-client` |
| MinerU Server URL | `http://mineru-vllm-server:30000` |
| Delete Output | 开启 |

> 必须选择 `vlm-http-client`，让 MinerU API 复用本部署独立的 vLLM Server。若选 `vlm-vllm-engine`，MinerU API 会在容器内再启动一个 vLLM，导致显存冲突。

### 5.2 知识库启用 MinerU

编辑知识库配置：

1. **Layout Recognize** 选择 `MinerU`
2. **MinerU Options**：
   - Parse Method: `auto`
   - Language: 按文档选择
   - Formula Recognition: 开启（如需公式）
   - Table Recognition: 开启（如需表格）
3. 保存并重新解析文档。

### 5.3 验证 GPU 解析

上传一个 PDF 后观察：

```bash
watch -n 2 nvidia-smi
```

---

## 6. 统一内存与显存优化（DGX Spark）

### 6.1 统一内存分配建议

DGX Spark 128 GB 统一内存推荐分配：

| 用途 | 建议内存 | 说明 |
|------|----------|------|
| 系统保留 | 16 GB | OS、Docker、缓存 |
| MinerU / vLLM | 32–64 GB | 根据并发和文档复杂度调整 |
| infinity-emb | 16 GB | GPU embedding/reranker |
| Elasticsearch | 32 GB | JVM heap + 文件缓存 |
| RAGFlow 服务 | 16 GB | 主容器、任务调度 |
| MySQL + Redis + MinIO | 8 GB | 元数据与缓存 |
| 预留扩展 | 16–32 GB | 后续扩容缓冲 |

### 6.2 vLLM 显存利用率调优

`tools/mineru-gpu/docker-compose.yml` 中 `mineru-vllm-server` 默认使用 `--gpu-memory-utilization 0.3`（约 36–38 GB）。如需调整：

```yaml
command:
  - --host
  - 0.0.0.0
  - --port
  - "30000"
  - --gpu-memory-utilization
  - "0.2"   # 约 24 GB，适合显存紧张场景
```

或提高到 `0.4`–`0.5` 以获得更高并发吞吐。

### 6.3 并发控制

MinerU API 默认限制并发请求数为 3。增大该值会线性增加显存占用。建议：

- 单用户/低并发：3
- 中等并发（5–10 人）：5–8
- 高并发：横向扩展 vLLM Server（多卡或多节点）

---

## 7. 常见问题

### 7.1 `mineru-api` 无法加入 `docker_ragflow` 网络

确保 RAGFlow 主服务已启动：

```bash
docker network ls | grep docker_ragflow
```

如果没有 `docker_ragflow`，检查 `docker-compose-dgx-arm64.yml` 是否已启动。

### 7.2 RAGFlow 报 `[ERROR]MinerU not found.`

按顺序排查：

1. RAGFlow 中 MinerU 模型实例是否已添加且名称与知识库配置一致。
2. `mineru-api` 是否健康：`curl http://localhost:8000/openapi.json`。
3. RAGFlow 容器能否访问 `mineru-api:8000`。
4. `mineru-api` 容器内能否访问 `mineru-vllm-server:30000`。
5. 查看日志：
   ```bash
   docker logs ragflow | grep -i mineru
   docker logs mineru-api
   docker logs mineru-vllm-server
   ```

### 7.3 显存不足

错误示例：

```text
Free memory on device cuda:0 (xx/xx GiB) on startup is less than desired GPU memory utilization
```

解决：

- 降低 `mineru-vllm-server` 的 `--gpu-memory-utilization`。
- 确认 RAGFlow 中 MinerU Backend 为 `vlm-http-client`，避免 API 再启一个 vLLM。
- 检查是否有其他进程占用 GPU 显存：`nvidia-smi`。

### 7.4 模型下载慢或失败

- 确认能访问 `modelscope.cn`。
- 可预先将模型下载到 `/home/d0ngfq/models/mineru/models/`，避免运行时下载。

### 7.5 `http://mineru-vllm-server:30000/health` 返回 405

vLLM `/health` 只支持 GET。RAGFlow 旧代码会用 HEAD 请求，当前项目已修复，会在 HEAD 失败后回退到 GET。如仍报错，请确保 `deepdoc/parser/mineru_parser.py` 包含 `_is_http_endpoint_valid` 的 GET fallback 逻辑。

---

## 8. 参考

- [RAGFlow DGX ARM64 部署指南](./deploy_dgx_arm64.md)
- [DGX Spark 统一内存架构优化方案](./dgx_spark_optimization_guide.md)
- [RAGFlow 生产环境运维手册](./operations_guide.md)
- [MinerU 官方 Docker 部署文档](https://opendatalab.github.io/MinerU/zh/quick_start/docker_deployment/)
- [MinerU GitHub](https://github.com/opendatalab/MinerU)
- `tools/mineru-gpu/README.md`
