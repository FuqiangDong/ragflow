# RAGFlow NVIDIA DGX ARM64 生产环境部署指南

> **版本**：RAGFlow v0.26.1  
> **适用对象**：NVIDIA DGX Spark等 ARM64 (aarch64) GPU 服务器  
> **目标环境**：生产环境（Production）  
> **最后更新**：2026-06-25

---

## 1. 重要声明与适用范围

### 1.1 官方镜像限制

RAGFlow 官方发布的 Docker 镜像（`infiniflow/ragflow:*`）**仅针对 x86_64 (amd64) 架构构建**，未提供官方 ARM64 镜像。当前仓库的 CI 构建脚本（`.github/workflows/release.yml`）没有使用 `docker buildx` 进行多架构构建，也没有 `--platform linux/arm64` 参数。

因此，在 **NVIDIA DGX ARM64** 环境部署生产服务时，必须：

1. **从源码自行构建 ARM64 镜像**。
2. **修改部分构建脚本和入口脚本**，修复当前代码库中仍存在的 x86_64 硬编码问题。
3. **替换或禁用部分 x86_64 独占依赖**（如 Chrome for Testing x86_64、ONNX Runtime GPU x86_64 wheel）。

### 1.2 本文档目标

本指南基于当前代码仓库（`v0.26.1`）的工程结构、Docker/Compose 配置和依赖清单，给出在 ARM64 DGX 上部署 RAGFlow 的**完整、可复现、可维护**的生产级方案。所有版本号、镜像名、依赖项均与仓库当前代码保持一致。

### 1.3 已知限制（ARM64 DGX）

| 能力 | x86_64 | ARM64 DGX | 说明 |
|------|--------|-----------|------|
| 官方预编译 Docker 镜像 | ✅ | ❌ | 必须自行构建 |
| CPU 模式 OCR/版面分析 | ✅ | ✅ | 使用 `onnxruntime==1.23.2` CPU wheel |
| GPU 模式 OCR（CUDAExecutionProvider） | ✅ | ❌ | `onnxruntime-gpu==1.23.2` 无官方 Linux/ARM64 wheel |
| Chrome / Web 抓取 / Browser Agent | ✅ | ⚠️ 需替换 | Chrome for Testing 无 `linux-arm64` 构建，需改用 Ubuntu `chromium` |
| Infinity 向量数据库（doc engine） | ✅ | ❌ 不建议 | `infiniflow/infinity:v0.7.0` 官方未正式支持 ARM64 |
| Elasticsearch 8.11.3 | ✅ | ✅ | 推荐作为向量引擎 |
| OpenSearch 2.19.1 | ✅ | ✅ | 备选 |
| TEI GPU Embedding | ✅ | ❌ 无官方镜像 | `text-embeddings-inference` 官方镜像无 `linux/arm64` tag |
| 独立 `infinity-emb` GPU 服务 | ✅ | ✅ | 推荐方案：在主容器外单独部署 GPU embedding/reranker |
| PyTorch CUDA wheel（Python 3.13） | ✅ | ❌ | PyTorch 官方未提供 `cp313` + `linux_aarch64` + CUDA 的 wheel |
| Go 后端（`API_PROXY_SCHEME=go/hybrid`） | ✅ | ✅ | 需确保 `office_oxide` 提供 `native-linux-aarch64` asset |

---

## 2. 硬件与软件前置条件

### 2.1 硬件要求

| 组件 | 生产最低要求 | DGX Spark 推荐配置 |
|------|--------------|-------------------|
| CPU | 64 核 ARM64 | GB10 Grace，20 核 Cortex-X4 |
| 内存 | 256 GB | 128 GB 统一内存（可支撑中小规模生产） |
| GPU | 可选；若启用，需 NVIDIA GPU + CUDA 12.x | Blackwell GPU（共享统一内存） |
| 存储 | SSD/NVMe，系统盘 200 GB，数据盘 ≥ 2 TB | 4 TB NVMe（建议系统 200 GB + 数据 3.8 TB） |
| 网络 | 10 Gbps 内网 | 10 GbE 或更高 |

### 2.2 软件要求（与仓库代码严格一致）

| 软件 | 版本要求 | 说明 |
|------|----------|------|
| 操作系统 | Ubuntu 22.04/24.04 LTS ARM64 | DGX OS 6.x 基于 Ubuntu，同样适用 |
| Linux Kernel | ≥ 5.15 | 建议 ≥ 6.2 以获得更好的容器与 GPU 支持 |
| Docker Engine | ≥ 24.0 | 必须支持 `docker buildx` 与 `compose` 插件 |
| Docker Compose | ≥ 2.20 | 用于 `deploy.resources.reservations.devices` 语法 |
| NVIDIA Driver | 与 CUDA 12.x 兼容 | 例如 535/550/560 系列数据中心驱动 |
| NVIDIA Container Toolkit | ≥ 2.17 | 必须启用 `nvidia-ctk` runtime |
| Docker Hub 镜像加速 | 国内环境必需 | 中科大/163/阿里云等，否则 `buildx`/`FROM` 拉取失败 |
| CUDA | 12.4 | 与 PyTorch `cu124` wheel 及 Blackwell 兼容 |
| CMake | ≥ 4.0 | 仓库 `internal/cpp/CMakeLists.txt` 要求 |
| Go | ≥ 1.26.4 | 由 `go.mod` 指定；若启用 Go 后端 |
| Python | 3.13 | 由 `pyproject.toml` 指定（`requires-python = ">=3.13,<3.14"`） |
| Node.js | 20.x | 由 `Dockerfile` 通过 NodeSource 安装 |
| uv | 0.9.16 | 由 `ragflow_deps/download_deps.py` 下载 |

### 2.3 核心依赖版本对照表

| 依赖/组件 | 版本 | 来源文件 |
|-----------|------|----------|
| RAGFlow | v0.26.1 | `pyproject.toml`、`docker/.env` |
| Elasticsearch | 8.11.3 | `docker/.env` |
| OpenSearch | 2.19.1 | `docker/docker-compose-base.yml` |
| Infinity doc engine | v0.7.0 | `docker/docker-compose-base.yml`（ARM64 不建议使用） |
| MySQL | 8.0.39 | `docker/docker-compose-base.yml` |
| MinIO | RELEASE.2026-03-25T00-00-00Z | `docker/docker-compose-base.yml`（`pgsty/minio`） |
| Redis/Valkey | 8 | `docker/docker-compose-base.yml` |
| NATS | 2.14.2 | `docker/docker-compose-base.yml` |
| ONNX Runtime | 1.23.2（CPU） | `pyproject.toml`、`uv.lock` |
| Infinity SDK | 0.7.0 | `pyproject.toml` |
| Infinity-emb | 0.0.66 | `pyproject.toml` |
| Tika Server | 3.3.0 | `ragflow_deps/download_deps.py` |
| libssl1.1 | 1.1.1f-1ubuntu2 | `ragflow_deps/download_deps.py` |
| Chrome for Testing | 121.0.6167.85（仅 x86_64） | `ragflow_deps/download_deps.py` |
| stagehand-server-v3 | v3.7.2 | `ragflow_deps/download_deps.py`、`go.mod` |
| office_oxide | 0.1.2 | `build.sh`、`.github/workflows/release.yml` |
| xgboost | 1.6.0 | `pyproject.toml` |
| Nginx | 1.31.0-1~noble | `Dockerfile` |

### 2.4 宿主机初始化检查

```bash
# 1. 确认架构
uname -m
# 预期输出：aarch64

# 2. 确认 NVIDIA 驱动与 CUDA 版本
nvidia-smi
nvidia-smi -q | grep "CUDA Version"
# 预期：CUDA Version >= 12.4

# 3. 确认 NVIDIA Container Toolkit 已配置
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# 4. 确认 GPU 容器可用
docker run --rm --gpus all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi

# 5. 确认 vm.max_map_count（Elasticsearch/OpenSearch 需要）
sysctl vm.max_map_count
# 若小于 262144，写入 /etc/sysctl.conf
sudo sysctl -w vm.max_map_count=262144

# 6. 确认 docker buildx 可用
docker buildx create --use --name ragflow-builder --driver docker-container || true
docker buildx inspect --bootstrap
```

### 2.5 Docker Hub 镜像加速（国内环境）

在中国大陆网络环境下，`docker buildx` 的 `docker-container` driver 需要拉取 `moby/buildkit` 镜像，`Dockerfile` 也需要拉取 `ubuntu:24.04`、`nvidia/cuda` 等基础镜像。若直接访问 Docker Hub 超时，请配置镜像加速：

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live",
    "https://hub.rat.dev"
  ]
}
EOF
sudo systemctl restart docker
```

> 若上述公共镜像站失效，建议使用阿里云容器镜像服务（ACR）个人版的专属加速地址。

**ARM64 宿主机可跳过 buildx**：因为宿主机本身就是 `linux/arm64`，直接使用原生 `docker build --platform linux/arm64 ...` 即可，无需 `docker-container` driver。这能避免 buildkit 镜像拉取问题。

---

## 3. 架构决策与最优配置

### 3.1 总体架构（推荐）

在 ARM64 DGX 上，推荐采用**“主容器 CPU + 独立 embedding GPU 服务”**的混合架构：

```text
┌─────────────────────────────────────────────────────────────┐
│                      DGX Spark (ARM64)                       │
│  ┌─────────────────────┐    ┌──────────────────────────┐   │
│  │ RAGFlow 主容器       │    │ infinity-emb GPU 服务     │   │
│  │ Python 3.13 / CPU    │◄──►│ Python 3.12 / CUDA 12.4  │   │
│  │ ONNX Runtime CPU     │    │ BGE-M3 + bge-reranker-v2 │   │
│  │ Chromium ARM64       │    │                          │   │
│  └─────────────────────┘    └──────────────────────────┘   │
│           │                            │                    │
│           ▼                            ▼                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Elasticsearch 8.11.3 / MySQL 8.0.39 / MinIO / Valkey │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

**设计理由**：

- RAGFlow 主镜像基于 Python 3.13，而 PyTorch 官方未提供 `cp313 + linux_aarch64 + CUDA` 的 wheel。强行在镜像内跑 GPU torch 会回退到 CPU或构建失败。
- `onnxruntime-gpu==1.23.2` 无 Linux/ARM64 wheel，DeepDoc OCR/版面分析在 ARM64 上只能走 CPU。
- 独立的 `infinity-emb` 服务基于 `nvcr.io/nvidia/pytorch:25.02-py3` 容器（内置 Python 3.12 + PyTorch 2.6.0 + CUDA 12.8），专门负责 embedding/reranker 的 GPU 加速，与主容器解耦。
- 该架构便于分别扩容、监控和升级。

> **关于 CUDA 驱动版本**：DGX Spark 可能预装支持 CUDA 13.0 的驱动（如 `nvidia-smi` 显示 `CUDA Version: 13.0`）。这表示驱动**最高**支持 CUDA 13.0，向下兼容 CUDA 12.x。PyTorch 官方 PyPI 未提供 `linux_aarch64` 的 CUDA wheel，因此不能通过 `pip install torch==x.x.x+cu124` 在 ARM64 上安装 GPU 版 PyTorch；必须基于 NVIDIA PyTorch 容器（如 `nvcr.io/nvidia/pytorch:25.02-py3`，内置 CUDA 12.8）来运行 GPU 推理。

### 3.2 向量数据库选择

在 ARM64 生产环境中，**强烈建议选用 Elasticsearch 8.11.3** 作为 `DOC_ENGINE`。

- ✅ Elasticsearch 官方镜像 `elasticsearch:8.11.3` 支持 `linux/arm64`。
- ✅ RAGFlow 对 Elasticsearch 的支持最成熟（默认文档引擎）。
- ❌ `infiniflow/infinity:v0.7.0` 官方未正式支持 ARM64，生产环境应避免使用。

```bash
# .env 中设置
DOC_ENGINE=elasticsearch
DEVICE=cpu
COMPOSE_PROFILES=elasticsearch,cpu
API_PROXY_SCHEME=python
```

### 3.3 GPU vs CPU 模式（主容器）

- 主容器 `DEVICE=cpu`：DeepDoc、版面分析、torch 运行时安装均走 CPU。这是 ARM64 下的稳定选择。
- 若强行 `DEVICE=gpu`：
  - `pip_install_torch()` 会安装 CPU wheel（PyPI 上 ARM64 cp313 只有 CPU wheel）。
  - `onnxruntime` 没有 CUDAExecutionProvider，OCR 仍会回退 CPU。
  - 因此主容器**不建议**设置 `DEVICE=gpu`。

GPU 算力通过独立的 `infinity-emb` 服务承载，见第 7 章。

### 3.4 API 后端模式

当前代码处于 Python 与 Go 后端并行阶段：

- `API_PROXY_SCHEME=python`（默认，最稳定，推荐生产首阶段使用）。
- `API_PROXY_SCHEME=go`：需要成功构建 Go 二进制；C++ tokenizer 与 `office_oxide` 的 ARM64 原生库必须就绪。
- `API_PROXY_SCHEME=hybrid`：风险最高，不建议首次生产部署使用。

本指南默认以 `python` 模式部署；Go 后端构建在附录中说明。

### 3.5 DGX Spark 128 GB 内存分配规划

| 组件 | 分配 | 用途 |
|------|------|------|
| **系统预留** | 16 GB | OS + Docker + 系统服务 |
| **infinity-emb GPU 服务** | 16 GB | BGE-M3 + bge-reranker-v2-m3（含 GPU/CPU 混用） |
| **DeepDoc / OCR** | 8 GB | 文档解析（PDF/图片，CPU 推理） |
| **Elasticsearch** | 32 GB | 向量与全文索引 |
| **RAGFlow 主服务** | 16 GB | API + 任务调度 |
| **MySQL + Valkey + MinIO** | 8 GB | 元数据 + 缓存 + 对象存储 |
| **预留（未来扩展）** | 32 GB | 扩展到更大文档规模 |

---

## 4. 源码级 ARM64 适配修改

在构建镜像之前，必须修复当前代码库中对 x86_64 的硬编码。以下提供一个**一键补丁脚本**，所有修改均可通过 `git diff` 复核。

### 4.1 创建补丁脚本

```bash
cat > patch_dgx_arm64.sh << 'EOF'
#!/bin/bash
set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$PROJECT_ROOT"

echo "=== 应用 DGX ARM64 适配补丁 ==="

# -----------------------------------------------------------------------------
# 1. Dockerfile：ARM64 下安装 chromium / chromium-driver
#    Chrome for Testing 121.0.6167.85 没有 linux-arm64 构建；
#    Ubuntu 24.04 官方 chromium 为 snap 过渡包，Docker 内改用 xtradeb PPA 原生 deb
# -----------------------------------------------------------------------------
if grep -q 'source=/chrome-linux64-121-0-6167-85,target=/chrome-linux64.zip' Dockerfile; then
    python3 << 'PYEOF'
import re

with open("Dockerfile", "r") as f:
    content = f.read()

old = '''# Add dependencies of selenium
RUN --mount=type=bind,from=infiniflow/ragflow_deps:latest,source=/chrome-linux64-121-0-6167-85,target=/chrome-linux64.zip \\
    unzip /chrome-linux64.zip && \\
    mv chrome-linux64 /opt/chrome && \\
    ln -s /opt/chrome/chrome /usr/local/bin/
RUN --mount=type=bind,from=infiniflow/ragflow_deps:latest,source=/chromedriver-linux64-121-0-6167-85,target=/chromedriver-linux64.zip \\
    unzip -j /chromedriver-linux64.zip chromedriver-linux64/chromedriver && \\
    mv chromedriver /usr/local/bin/ && \\
    rm -f /usr/bin/google-chrome'''

new = '''# Add dependencies of selenium (architecture-aware)
RUN --mount=type=bind,from=infiniflow/ragflow_deps:latest,source=/,target=/deps \\
    set -eux; \\
    arch="$(uname -m)"; \\
    if [ "$arch" = "aarch64" ]; then \\
        # Ubuntu 24.04 官方 chromium 为 snap 过渡包，Docker 内无法使用；
        # 使用 xtradeb PPA 安装原生 ARM64 deb（生产环境请评估第三方源风险）
        apt-get update && \\
        apt-get install -y --no-install-recommends ca-certificates gnupg software-properties-common && \\
        add-apt-repository -y ppa:xtradeb/apps && \\
        printf '%s\\n' \\
          'Package: *' \\
          'Pin: release o=LP-PPA-xtradeb-apps' \\
          'Pin-Priority: 100' \\
          '' \\
          'Package: chromium*' \\
          'Pin: release o=LP-PPA-xtradeb-apps' \\
          'Pin-Priority: 700' > /etc/apt/preferences.d/chromium-xtradeb && \\
        apt-get update && \\
        apt-get install -y --no-install-recommends chromium chromium-driver && \\
        ln -sf /usr/bin/chromium /usr/local/bin/chrome && \\
        ln -sf /usr/bin/chromedriver /usr/local/bin/chromedriver && \\
        rm -f /usr/bin/google-chrome && \\
        rm -rf /var/lib/apt/lists/*; \\
    else \\
        unzip /deps/chrome-linux64-121-0-6167-85 && \\
        mv chrome-linux64 /opt/chrome && \\
        ln -s /opt/chrome/chrome /usr/local/bin/; \\
        unzip -j /deps/chromedriver-linux64-121-0-6167-85 chromedriver-linux64/chromedriver && \\
        mv chromedriver /usr/local/bin/ && \\
        rm -f /usr/bin/google-chrome; \\
    fi'''

if old not in content:
    raise RuntimeError("Dockerfile 内容不符合预期，请手动检查")

content = content.replace(old, new)
with open("Dockerfile", "w") as f:
    f.write(content)
print("已修补 Dockerfile")
PYEOF
else
    echo "Dockerfile 已修补或内容变化，跳过"
fi

# -----------------------------------------------------------------------------
# 2. entrypoint.sh：LD_LIBRARY_PATH 改为架构感知
# -----------------------------------------------------------------------------
sed -i 's|export LD_LIBRARY_PATH="/usr/lib/x86_64-linux-gnu/"|export LD_LIBRARY_PATH="/usr/lib/$(uname -m)-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"|' docker/entrypoint.sh || true

# -----------------------------------------------------------------------------
# 3. launch_backend_service.sh：LD_LIBRARY_PATH 改为架构感知
# -----------------------------------------------------------------------------
sed -i 's|export LD_LIBRARY_PATH=/usr/lib/x86_64-linux-gnu/|export LD_LIBRARY_PATH="/usr/lib/$(uname -m)-linux-gnu${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"|' docker/launch_backend_service.sh || true

echo "=== 补丁应用完成 ==="
echo "建议执行：git diff -- Dockerfile docker/entrypoint.sh docker/launch_backend_service.sh"
EOF

chmod +x patch_dgx_arm64.sh
```

### 4.2 执行补丁

```bash
cd /path/to/ragflow
./patch_dgx_arm64.sh
```

### 4.3 手动复核要点

执行补丁后，请确认以下三处已变更：

1. `Dockerfile` 中 Chrome 安装段改为按 `uname -m` 判断，ARM64 下安装 `chromium` + `chromium-driver`。
2. `docker/entrypoint.sh` 中 `LD_LIBRARY_PATH` 不再硬编码 `x86_64-linux-gnu`。
3. `docker/launch_backend_service.sh` 中 `LD_LIBRARY_PATH` 不再硬编码 `x86_64-linux-gnu`。

```bash
git diff -- Dockerfile docker/entrypoint.sh docker/launch_backend_service.sh
```

---

## 5. 构建 ARM64 镜像

### 5.1 准备源码与依赖

```bash
# 1. 克隆仓库并切换到 v0.26.1
git clone https://github.com/infiniflow/ragflow.git
cd ragflow
git checkout v0.26.1

# 2. 应用第 4 节的 ARM64 补丁
./patch_dgx_arm64.sh

# 3. 下载依赖资源（含 x86_64 的 Chrome zip；ARM64 下仅作为占位，不会被使用）
cd ragflow_deps
# 国内网络环境使用：uv run download_deps.py --china-mirrors
uv run download_deps.py

# 4. 构建依赖镜像（必须在 ARM64 宿主机上执行）
# 注意：不存在 Dockerfile.deps；依赖镜像 Dockerfile 位于 ragflow_deps/Dockerfile
docker build --platform linux/arm64 -f Dockerfile -t infiniflow/ragflow_deps:latest .
cd ..
```

> **常见问题**：`uv run download_deps.py --china-mirrors` 在下载 `stagehand-server-v3-linux-*` 时可能因 GitHub 连接 reset 失败。请手动通过 `ghproxy.com` 或其他可用代理下载后重新运行脚本：
> ```bash
> cd ragflow_deps
> curl -fsSL -o stagehand-server-v3-linux-x64 https://ghproxy.com/https://github.com/browserbase/stagehand/releases/download/stagehand-server-v3/v3.7.2/stagehand-server-v3-linux-x64
> curl -fsSL -o stagehand-server-v3-linux-arm64 https://ghproxy.com/https://github.com/browserbase/stagehand/releases/download/stagehand-server-v3/v3.7.2/stagehand-server-v3-linux-arm64
> cd .. && uv run ragflow_deps/download_deps.py --china-mirrors
> ```

### 5.2 构建 RAGFlow 主镜像（ARM64）

```bash
# 方案 A：原生 docker build（推荐，宿主机本身就是 ARM64，无需 buildx）
docker build \
  --platform linux/arm64 \
  --build-arg NEED_MIRROR=1 \
  -f Dockerfile \
  -t infiniflow/ragflow:nightly .

# 方案 B：buildx 多架构构建（需要能稳定拉取 moby/buildkit 镜像）
docker buildx create --use --name ragflow-builder --driver docker-container || true
docker buildx inspect --bootstrap

docker buildx build \
  --platform linux/arm64 \
  --build-arg NEED_MIRROR=1 \
  -f Dockerfile \
  -t <your-registry>/ragflow:v0.26.1-arm64 \
  --push .
```

参数说明：

- `--platform linux/arm64`：强制构建 ARM64 镜像。
- `--build-arg NEED_MIRROR=0`：海外网络；国内环境改为 `1`。
- `-t <your-registry>/ragflow:v0.26.1-arm64`：替换为你的私有镜像仓库。
- `--push`：直接推送；如仅需本地加载，改为 `--load`（单架构可用）。

### 5.3 验证镜像架构

```bash
docker pull <your-registry>/ragflow:v0.26.1-arm64
docker inspect --format='{{.Os}}/{{.Architecture}}' <your-registry>/ragflow:v0.26.1-arm64
# 预期输出：linux/arm64
```

---

## 6. 部署目录准备与 `.env` 配置

### 6.1 准备部署目录

```bash
mkdir -p /opt/ragflow-prod
cd /opt/ragflow-prod

# 复制 Docker/Compose 与配置模板
cp /path/to/ragflow/docker/docker-compose.yml .
cp /path/to/ragflow/docker/docker-compose-base.yml .
cp /path/to/ragflow/docker/.env .
cp /path/to/ragflow/docker/service_conf.yaml.template .
cp /path/to/ragflow/docker/entrypoint.sh .
cp /path/to/ragflow/docker/init.sql .

# 创建持久化目录
mkdir -p ragflow-logs esdata01 mysql_data minio_data redis_data infinity-emb-cache
```

### 6.2 关键 `.env` 配置（最优方案）

```bash
# ---------------------------------------------------------------------------
# 基础部署模式
# ---------------------------------------------------------------------------
DOC_ENGINE=elasticsearch
DEVICE=cpu                       # 主容器在 ARM64 下必须保持 cpu
COMPOSE_PROFILES=elasticsearch,cpu
API_PROXY_SCHEME=python          # 生产首阶段推荐 python

# ---------------------------------------------------------------------------
# RAGFlow 镜像
# ---------------------------------------------------------------------------
RAGFLOW_IMAGE=<your-registry>/ragflow:v0.26.1-arm64

# ---------------------------------------------------------------------------
# 端口映射（根据现场网络策略调整）
# ---------------------------------------------------------------------------
SVR_WEB_HTTP_PORT=80
SVR_WEB_HTTPS_PORT=443
SVR_HTTP_PORT=9380
ADMIN_SVR_HTTP_PORT=9381
SVR_MCP_PORT=9382
GO_HTTP_PORT=9384
GO_ADMIN_PORT=9383

# ---------------------------------------------------------------------------
# Elasticsearch
# ---------------------------------------------------------------------------
STACK_VERSION=8.11.3
ES_HOST=es01
ES_PORT=1200
ELASTIC_PASSWORD=<生成强密码>

# ---------------------------------------------------------------------------
# MySQL
# ---------------------------------------------------------------------------
MYSQL_HOST=mysql
MYSQL_PORT=3306
EXPOSE_MYSQL_PORT=3306
MYSQL_DBNAME=rag_flow
MYSQL_PASSWORD=<生成强密码>
MYSQL_MAX_PACKET=1073741824

# ---------------------------------------------------------------------------
# MinIO
# ---------------------------------------------------------------------------
MINIO_HOST=minio
MINIO_PORT=9000
MINIO_CONSOLE_PORT=9001
MINIO_USER=rag_flow
MINIO_PASSWORD=<生成强密码，至少 8 位>

# ---------------------------------------------------------------------------
# Valkey（Redis 兼容）
# ---------------------------------------------------------------------------
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=<生成强密码>

# ---------------------------------------------------------------------------
# 资源限制
# ---------------------------------------------------------------------------
MEM_LIMIT=34359738368            # 32 GB，根据 DGX 内存调整
TZ=Asia/Shanghai
THREAD_POOL_MAX_WORKERS=128
DOC_BULK_SIZE=4
EMBEDDING_BATCH_SIZE=16

# ---------------------------------------------------------------------------
# 安全与功能开关
# ---------------------------------------------------------------------------
REGISTER_ENABLED=1
DISABLE_PASSWORD_LOGIN=false
USE_DOCLING=false

# ---------------------------------------------------------------------------
# infinity-emb GPU 服务
# ---------------------------------------------------------------------------
INFINITY_EMB_IMAGE=infiniflow/infinity-emb:gpu
INFINITY_EMB_PORT=8080

# ---------------------------------------------------------------------------
# TEI（ARM64 下不启用，仅用于消除 compose 警告）
# ---------------------------------------------------------------------------
TEI_IMAGE_CPU=ghcr.io/huggingface/text-embeddings-inference:cpu-1.6
TEI_IMAGE_GPU=ghcr.io/huggingface/text-embeddings-inference:1.6
TEI_MODEL=BAAI/bge-m3

# ---------------------------------------------------------------------------
# HuggingFace 镜像（国内环境）
# ---------------------------------------------------------------------------
HF_ENDPOINT=https://hf-mirror.com
```

> **注意**：`TEI_IMAGE_CPU` / `TEI_IMAGE_GPU` 在 ARM64 下不建议启用，因为官方 TEI 镜像无 `linux/arm64` 版本。embedding/reranker 改由第 7 章的独立 `infinity-emb` GPU 服务提供。

### 6.3 关键安全加固

1. **替换默认密码**：`MYSQL_PASSWORD`、`ELASTIC_PASSWORD`、`MINIO_PASSWORD`、`REDIS_PASSWORD` 必须使用强密码。
2. **替换 RSA 密钥对**：
   ```bash
   openssl genrsa -out conf/private.pem 2048
   openssl rsa -in conf/private.pem -pubout -out conf/public.pem
   ```
3. **关闭公网端口**：仅暴露 `80/443`（或指定反向代理端口）到公网；MySQL、ES、MinIO Console、Valkey 必须仅在内网或本机暴露。
4. **启用 HTTPS**：
   - 在 `docker/nginx/` 目录准备 `ssl.crt` 与 `ssl.key`。
   - 挂载自定义 Nginx 配置，启用 TLS 1.2/1.3。
5. **备份策略**：配置定时备份 MySQL、ES 快照、MinIO bucket。

---

## 7. 独立 infinity-emb GPU 服务（Embedding + Reranker）

由于主容器在 ARM64 下无法使用 GPU torch，推荐单独部署一个基于 **NVIDIA PyTorch 容器** 的 `infinity-emb` 服务，负责所有 embedding 和 reranker 的 GPU 推理。DGX Spark 的 GB10 (Blackwell) 需要 **NVIDIA PyTorch 25.02**（CUDA 12.8）才能被正确识别。

### 7.1 创建 infinity-emb Dockerfile

```bash
cat > /opt/ragflow-prod/Dockerfile.infinity-emb << 'EOF'
FROM nvcr.io/nvidia/pytorch:25.02-py3

ENV DEBIAN_FRONTEND=noninteractive
ENV PYTHONDONTWRITEBYTECODE=1
ENV PIP_NO_CACHE_DIR=1
ENV HF_HOME=/data/hf_cache

# NVIDIA PyTorch 25.02 已预装 Python 3.12 + PyTorch 2.7.0 + CUDA 12.8，
# 原生支持 Blackwell / GB10。以下安装均用 --no-deps 避免覆盖 CUDA torch。

RUN pip install --no-cache-dir --upgrade pip setuptools wheel

# torch-dependent packages: 用 --no-deps 防止 pip 安装 CPU-only torch
RUN pip install --no-cache-dir --no-deps \
    'infinity-emb==0.0.66' \
    'sentence-transformers==3.2.1' \
    'transformers==4.46.3'

# Runtime dependencies（均不依赖 torch）
RUN pip install --no-cache-dir \
    numpy scipy scikit-learn nltk sentencepiece \
    'huggingface-hub==0.25.2' 'tokenizers==0.20.3' safetensors tqdm \
    'regex>=2025.10.22' filetype packaging pyyaml requests \
    fastapi uvicorn pydantic typer rich Pillow orjson aiohttp \
    einops python-multipart protobuf \
    prometheus-fastapi-instrumentator

# optimum 1.23.3 仍包含 optimum.bettertransformer；新版本已移除
RUN pip install --no-cache-dir --no-deps 'optimum==1.23.3' && \
    pip install --no-cache-dir coloredlogs sympy

# 验证 torch 仍编译了 CUDA（build 阶段无 GPU，不检查 cuda.is_available）
RUN python3 - <<'PY'
import torch
assert torch.version.cuda is not None, "torch is CPU-only"
print("torch version:", torch.__version__)
print("torch built with CUDA:", torch.version.cuda)
PY

EXPOSE 8080

CMD ["infinity_emb", "v2", \
     "--model-id", "BAAI/bge-m3", \
     "--model-id", "BAAI/bge-reranker-v2-m3", \
     "--host", "0.0.0.0", \
     "--port", "8080", \
     "--device", "cuda", \
     "--batch-size", "128", \
     "--url-prefix", "/v1"]
EOF
```

### 7.2 构建并运行 infinity-emb

```bash
cd /opt/ragflow-prod

# 构建镜像（必须在 ARM64 宿主机上执行）
docker build --platform linux/arm64 -f Dockerfile.infinity-emb -t infiniflow/infinity-emb:gpu .

# 方式一：单独运行 GPU 服务
docker run -d --name infinity-emb \
  --gpus all \
  --restart unless-stopped \
  -p 8080:8080 \
  -v ./infinity-emb-cache:/data/hf_cache \
  -e HF_HOME=/data/hf_cache \
  infiniflow/infinity-emb:gpu

# 方式二：通过 docker compose 统一管理（推荐，见 7.4）

# 查看日志
docker logs -f infinity-emb

# 验证健康
curl -s http://localhost:8080/health
# 预期：{"unix":...}

# 验证 RAGFlow 实际调用的 /v1/embeddings
curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"BAAI/bge-m3","input":["hello"]}'
# 预期：200
```

> **注意**：首次启动会从 HuggingFace 下载 `BAAI/bge-m3` 与 `BAAI/bge-reranker-v2-m3`，耗时数分钟到数十分钟。国内环境建议在 compose 中为服务设置 `HF_ENDPOINT=https://hf-mirror.com`。

### 7.3 在 RAGFlow 中配置 Embedding 与 Reranker

进入 RAGFlow Web UI：**Model Provider** → **Add Model**。

**Embedding 模型**：

| 字段 | 值 |
|------|-----|
| Provider | `OpenAI-API-Compatible` |
| Model Name | `BAAI/bge-m3` |
| Base URL | `http://infinity-emb:8080` |
| API Key | 任意非空字符串（如 `dummy`） |
| Max Tokens | `8192`（BGE-M3 支持 8192 token；RAGFlow 内置 `BuiltinEmbed.MAX_TOKENS` 为 8000，可保持一致） |

**Reranker 模型**：

| 字段 | 值 |
|------|-----|
| Provider | `OpenAI-API-Compatible` |
| Model Name | `BAAI/bge-reranker-v2-m3` |
| Base URL | `http://infinity-emb:8080` |
| API Key | 任意非空字符串 |

> **关键**：RAGFlow 的 `OpenAI-API-Compatible` Provider 会在 Base URL 后自动拼接 `/v1`，即实际请求地址为 `http://infinity-emb:8080/v1/embeddings` 与 `http://infinity-emb:8080/v1/rerank`。因此 `infinity-emb` 必须以 `--url-prefix /v1` 启动（已写入 Dockerfile 与 compose 示例），否则会出现 `404 - {'detail': 'Not Found'}`。

### 7.4 通过 docker-compose 统一管理（推荐）

在 RAGFlow 源码 `docker/` 目录下创建 `docker-compose-dgx-arm64.yml`，直接 include 官方 `docker-compose-base.yml` 并添加 `ragflow` 与 `infinity-emb` 服务：

```yaml
# docker/docker-compose-dgx-arm64.yml
include:
  - ./docker-compose-base.yml

services:
  ragflow:
    depends_on:
      mysql:
        condition: service_healthy
      es01:
        condition: service_healthy
    image: ${RAGFLOW_IMAGE:-infiniflow/ragflow:nightly}
    container_name: ragflow
    command:
      - --enable-adminserver
      - --init-model-provider-tables
    ports:
      - ${SVR_WEB_HTTP_PORT:-80}:80
      - ${SVR_WEB_HTTPS_PORT:-443}:443
      - ${SVR_HTTP_PORT:-9380}:9380
      - ${ADMIN_SVR_HTTP_PORT:-9381}:9381
      - ${SVR_MCP_PORT:-9382}:9382
      - ${GO_HTTP_PORT:-9384}:9384
      - ${GO_ADMIN_PORT:-9383}:9383
    volumes:
      - ./ragflow-logs:/ragflow/logs
      - ./service_conf.yaml.template:/ragflow/conf/service_conf.yaml.template
      - ./entrypoint.sh:/ragflow/entrypoint.sh
    env_file: .env
    environment:
      - DEVICE=cpu
    networks:
      - ragflow
    restart: unless-stopped
    extra_hosts:
      - "host.docker.internal:host-gateway"

  infinity-emb:
    image: ${INFINITY_EMB_IMAGE:-infiniflow/infinity-emb:gpu}
    container_name: infinity-emb
    hostname: infinity-emb
    ports:
      - ${INFINITY_EMB_PORT:-8080}:8080
    env_file: .env
    environment:
      - DEVICE=cuda
      - HF_HOME=/data/hf_cache
    volumes:
      - ./infinity-emb-cache:/data/hf_cache
    networks:
      - ragflow
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

启动命令：

```bash
cd /path/to/ragflow/docker

# 启动 ES + MySQL + MinIO + Redis + RAGFlow + infinity-emb
docker compose -f docker-compose-dgx-arm64.yml --profile elasticsearch up -d
```

> 该 compose 文件默认不启用 `ragflow-go`、`sandbox`、`tei` 等 profile，保持最小可用集合。

### 7.5 可选：GPU 加速 PDF 解析（MinerU）

RAGFlow 默认的 DeepDoc PDF 解析/OCR 在 ARM64 上只能跑 CPU（`onnxruntime-gpu` 无 Linux/ARM64 wheel）。如果需要 GPU 加速 PDF 解析，可单独部署 **MinerU** 服务。

#### 推荐部署路径

在 DGX Spark 上，**已验证的部署路径为 `tools/mineru-gpu/`**：

- 采用 **独立 vLLM 后端 + MinerU HTTP API** 架构。
- 模型持久化到 `/home/d0ngfq/models/mineru/models/`。
- RAGFlow 中使用 `vlm-http-client` backend，避免 API 自己再启动 vLLM 导致显存冲突。

完整步骤见《[RAGFlow + MinerU GPU 部署指南（DGX Spark / ARM64 / Blackwell）](./deploy_mineru_gpu_dgx_arm64.md)》。

#### 快速接入

确保 RAGFlow 主服务已启动并创建了 `docker_ragflow` 网络后：

```bash
cd /path/to/ragflow/tools/mineru-gpu

# 构建镜像（DGX Spark/Blackwell 请编辑 Dockerfile 改用 nightly 基础镜像）
docker build -t mineru:latest -f Dockerfile .

# 启动独立 vLLM + API
docker compose up -d
```

#### 在 RAGFlow 中配置

进入 **User Settings -> Model Provider -> MinerU**：

| 字段 | 推荐值 |
|------|--------|
| MinerU API Server | `http://mineru-api:8000` |
| MinerU Backend | `vlm-http-client` |
| MinerU Server URL | `http://mineru-vllm-server:30000` |
| Delete Output | 开启 |

> 仓库根目录的 `Dockerfile.mineru-dgx-arm64` 和 `docker/docker-compose-mineru-dgx-arm64.yml` 是早期单容器方案，**未在 DGX Spark 实测**，不建议作为主要生产路径。

---

## 8. 启动 RAGFlow 服务

### 8.1 使用专用 compose 一键启动

```bash
cd /path/to/ragflow/docker

# 启动 ES + MySQL + MinIO + Redis + RAGFlow + infinity-emb
docker compose -f docker-compose-dgx-arm64.yml --profile elasticsearch up -d

# 等待基础设施就绪（约 1-3 分钟）
docker compose -f docker-compose-dgx-arm64.yml logs -f mysql es01 minio redis
```

### 8.2 查看各服务日志

```bash
# RAGFlow 主容器
docker compose -f docker-compose-dgx-arm64.yml logs -f ragflow

# infinity-emb GPU 服务
docker logs -f infinity-emb
```

### 8.3 验证服务健康

```bash
# 1. 检查容器状态
docker compose ps

# 2. 检查 API 健康接口
curl -s http://localhost:9380/api/v1/system/healthz | python3 -m json.tool

# 3. 检查 Web 访问
curl -s -o /dev/null -w "%{http_code}\n" http://localhost/
# 预期：200

# 4. 检查 ES 集群健康
curl -u elastic:<ELASTIC_PASSWORD> http://localhost:1200/_cluster/health

# 5. 检查 MinIO 健康
curl -f http://localhost:9000/minio/health/live

# 6. 检查 Valkey
docker compose exec redis redis-cli -a <REDIS_PASSWORD> ping

# 7. 检查 infinity-emb
curl -s http://localhost:8080/health

# 8. 检查 /v1/embeddings 是否可用（RAGFlow 实际调用的路径）
curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST http://localhost:8080/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"BAAI/bge-m3","input":["hello world"]}'
# 预期：200

# 9. 检查 /v1/rerank 是否可用
curl -s -o /dev/null -w "%{http_code}\n" \
  -X POST http://localhost:8080/v1/rerank \
  -H "Content-Type: application/json" \
  -d '{"model":"BAAI/bge-reranker-v2-m3","query":"hello","documents":["world"]}'
# 预期：200
```

首次启动时，`infinity-emb` 会下载并 warm-up 模型，日志中应出现类似：

```text
INFO infinity_emb: Starting model BAAI/bge-m3
INFO infinity_emb: Warm-up complete for BAAI/bge-m3
INFO infinity_emb: Starting model BAAI/bge-reranker-v2-m3
INFO infinity_emb: Warm-up complete for BAAI/bge-reranker-v2-m3
INFO:     Uvicorn running on http://0.0.0.0:8080
```

warm-up 完成后，`/health` 返回 HTTP 200：

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/health
# 预期：200
```

同时确认 GPU 被占用：

```bash
watch -n 1 nvidia-smi
```

应看到 `python` 进程占用了显存（如 `BAAI/bge-m3` 约 2–3 GB，`BAAI/bge-reranker-v2-m3` 约 2 GB）。

---

## 9. 性能调优（DGX Spark / Blackwell GPU）

### 9.1 infinity-emb Batch Size 调优

根据 GPU 显存（DGX Spark 共享内存约 128 GB）调整 `--batch-size`：

| GPU 可用内存 | 推荐 `--batch-size` | 说明 |
|-------------|---------------------|------|
| 16 GB | 32 | 过渡方案/轻量模型 |
| 32 GB | 64-128 | 中等规模 |
| 64 GB+ | 128-256 | DGX Spark 推荐 |

观察 `nvidia-smi` 显存占用与推理延迟，逐步提升 batch size。

### 9.2 DeepDoc / OCR CPU 线程调优

主容器 OCR 走 CPU，建议通过环境变量控制线程数，避免任务执行器之间过度抢占：

```bash
OCR_INTRA_OP_NUM_THREADS=2
OCR_INTER_OP_NUM_THREADS=2
THREAD_POOL_MAX_WORKERS=128
```

### 9.3 内存与系统优化

```bash
# 启用 Transparent Huge Pages
echo always > /sys/kernel/mm/transparent_hugepage/enabled

# 降低 swappiness（避免 swap）
sysctl vm.swappiness=1

# Elasticsearch 必须设置
sudo sysctl -w vm.max_map_count=262144
```

### 9.4 RAGFlow 任务执行器扩容

单容器内增加任务执行器进程数：

```yaml
# docker-compose.yml 中 ragflow-cpu 的 command
command:
  - --enable-adminserver
  - --init-model-provider-tables
  - --workers=4
```

多实例部署时，为每个实例指定唯一 `--host-id`，避免任务重复消费：

```yaml
command:
  - --enable-adminserver
  - --init-model-provider-tables
  - --workers=2
  - --host-id=dgx-node-01
```

---

## 10. 监控与运维

### 10.1 关键日志路径

```bash
# RAGFlow 主容器日志
docker compose logs -f ragflow-cpu

# infinity-emb 日志
docker logs -f infinity-emb

# Elasticsearch 日志
docker compose logs -f es01

# 系统日志
journalctl -u docker.service -f
```

### 10.2 健康检查脚本

创建 `/opt/ragflow-scripts/health_check.sh`：

```bash
#!/bin/bash
set -e

echo "=== DGX Spark RAGFlow 部署健康检查 ==="
echo "时间：$(date)"
echo ""

# 1. RAGFlow API
echo "1. RAGFlow API 健康："
curl -s http://localhost:9380/api/v1/system/healthz | python3 -m json.tool || echo "❌ RAGFlow API 异常"
echo ""

# 2. infinity-emb
echo "2. infinity-emb 服务状态："
curl -s http://localhost:8080/health | python3 -m json.tool || echo "❌ infinity-emb 异常"
echo ""

# 3. GPU 状态
echo "3. GPU 状态："
nvidia-smi --query-gpu=index,name,temperature.gpu,utilization.gpu,memory.used,memory.total --format=csv
echo ""

# 4. 内存使用
echo "4. 内存使用："
free -h
echo ""

# 5. ES 健康
echo "5. Elasticsearch 健康："
curl -s -u elastic:${ELASTIC_PASSWORD} http://localhost:1200/_cluster/health | python3 -m json.tool
echo ""

echo "=== 检查完成 ==="
```

赋予执行权限并运行：

```bash
chmod +x /opt/ragflow-scripts/health_check.sh
ELASTIC_PASSWORD=<your-password> /opt/ragflow-scripts/health_check.sh
```

---

## 11. 高可用与扩展

### 11.1 文档引擎高可用

- **Elasticsearch**：使用官方 ES 集群模式，至少 3 个 master-eligible 节点 + 多个 data 节点。
- **MySQL**：使用主从复制、MGR 或外部托管 MySQL（如 AWS RDS、阿里云 PolarDB）。
- **MinIO**：使用分布式 MinIO 集群（4 节点起步）或外部 S3/OSS。
- **Valkey**：使用 Sentinel 或 Cluster；RAGFlow 目前使用单实例，需自行验证集群模式兼容性。

### 11.2 应用层扩展

- `infinity-emb` 可水平扩展为多实例，前端用 Nginx 负载均衡。
- RAGFlow 主容器可水平扩展，任务执行器通过 `--host-id` 区分。

---

## 12. 故障排查

### 12.1 容器启动后立即退出

```bash
# 查看日志
docker compose logs --tail 200 ragflow-cpu

# 常见原因：
# 1. LD_LIBRARY_PATH 错误导致 .so 加载失败
# 2. service_conf.yaml.template 渲染失败（环境变量未正确传递）
# 3. MySQL/ES 未就绪，健康检查失败
```

### 12.2 Docker Hub / GitHub / HuggingFace 拉取失败

DGX Spark 在中国大陆网络环境下常见。**必须** 配置 Docker Hub 国内镜像：

```bash
# /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1panel.live",
    "https://hub.rat.dev",
    "https://docker.mirrors.ustc.edu.cn",
    "https://dockerpull.cn",
    "https://docker.hlmirror.com"
  ]
}
```

```bash
sudo systemctl restart docker
```

若仍无法获取基础镜像，可通过能访问外网的 x86_64 / ARM64 机器拉取后 `docker save` + `docker load` 到 DGX Spark。

### 12.3 构建 RAGFlow 主镜像时 `Dockerfile.deps` 不存在 / buildkit 拉取失败

RAGFlow v0.26.1 的依赖镜像构建文件实际路径为 `ragflow_deps/Dockerfile`，而非 `docker/Dockerfile.deps`。完整命令为：

```bash
cd /path/to/ragflow

# 1. 构建依赖数据镜像（无需 buildx，原生 docker build 即可）
docker build -f ragflow_deps/Dockerfile -t infiniflow/ragflow_deps:latest .

# 2. 构建主镜像（ARM64，原生 docker build）
docker build -f Dockerfile -t infiniflow/ragflow:nightly .
```

- 必须设置 `NEED_MIRROR=1`（作为 `ARG` 传入），否则容器内 apt/pip/resource 拉取会失败。
- 镜像较大，首次构建需要 30–90 分钟，请保证磁盘空间 > 60 GB。
- `ragflow_deps` 镜像包含双架构预编译文件，原生 `docker build` 在 ARM64 主机上会自动选择对应架构。

### 12.4 `Exec format error`

表示容器内存在 x86_64 二进制在 ARM64 上运行。检查：

- Chrome / ChromeDriver 是否为 ARM64 版本（应使用 `chromium` 系统包）。
- `ragflow_deps` 镜像是否正确构建了 ARM64 版本。
- `entrypoint.sh` 中是否硬编码了 x86_64 路径。

### 12.5 ES 启动失败 `max_map_count`

```bash
sudo sysctl -w vm.max_map_count=262144
# 永久生效：写入 /etc/sysctl.conf
```

### 12.6 GPU 不可用 / GB10 不被识别

DGX Spark 的 NVIDIA GB10 (Blackwell) 需要 **CUDA 12.8+ 驱动支持**。若使用 `nvcr.io/nvidia/pytorch:24.10-py3` 或更早镜像，`infinity-emb` 会报 `No supported GPU(s) detected`。必须使用 **25.02 及以上** 的 NVIDIA PyTorch 容器：

```dockerfile
FROM nvcr.io/nvidia/pytorch:25.02-py3
```

排查命令：

```bash
# 确认宿主机能识别 GPU
nvidia-smi

# 确认容器内能识别 GPU
docker run --rm --gpus all nvcr.io/nvidia/pytorch:25.02-py3 nvidia-smi

# 确认 infinity-emb 容器内能识别 GPU
docker exec infinity-emb nvidia-smi
```

若宿主机 `nvidia-smi` 显示 `CUDA Version: 13.0`，这是驱动支持的最高 CUDA 版本，不影响运行 CUDA 12.x 的容器。580.x 驱动向后兼容 cu124/cu128。

### 12.7 infinity-emb 镜像构建失败：torch / tokenizers / optimum / huggingface_hub 版本冲突

构建 GPU 版 `infinity-emb` 时，不要使用 PyPI 的 `torch==2.5.0+cu124`，因为 PyTorch 官方未提供 Linux ARM64 CUDA wheel。正确做法：

1. 基于 `nvcr.io/nvidia/pytorch:25.02-py3`（已含 CUDA ARM64 PyTorch）。
2. 安装 `infinity-emb` 及其依赖时加 `--no-deps`，防止 PyPI CPU-only torch 覆盖预装 CUDA torch。
3. 显式固定关键版本：
   - `sentence-transformers==3.2.1`
   - `transformers==4.46.3`
   - `tokenizers==0.20.3`
   - `huggingface-hub==0.25.2`
   - `optimum==1.23.3`（保留 bettertransformer）
4. 补齐运行依赖：`numpy scipy scikit-learn nltk sentencepiece` 等。

参考完整 Dockerfile 见 [8.2 构建镜像](#82-构建镜像)。

### 12.8 infinity-emb 启动成功但 `/health` 失败 / warm-up 失败

常见原因与对策：

1. **模型未下载完成**：首次启动会联网下载 `BAAI/bge-m3` 与 `BAAI/bge-reranker-v2-m3`。国内设置 `HF_ENDPOINT=https://hf-mirror.com`，并检查模型缓存目录是否持久化。
2. **GPU 不被识别**：见 [12.6 GPU 不可用](#126-gpu-不可用--gb10-不被识别)。
3. **监听地址错误**：必须加 `--host 0.0.0.0`，否则只能本机访问。
4. **DNS 解析失败**：容器内无法解析域名。检查 Docker daemon DNS 配置，或在 compose 中显式指定 `dns: [8.8.8.8, 114.114.114.114]`。

### 12.9 embedding/reranker 返回 404 或格式错误

错误示例：

```text
Fail to bind embedding model: Embedding request failed for OpenAI_APIEmbed.
Error: Error code: 404 - {'detail': 'Not Found'}
```

根因：RAGFlow 的 `OpenAI_APIEmbed` 会把 Base URL 与 `/v1` 拼接，再调用 `/v1/embeddings`；而 `infinity-emb` 默认暴露 `/embeddings`，没有 `/v1` 前缀。

解决方案：

1. 确认 `infinity-emb` 启动参数包含 `--url-prefix /v1`：

   ```bash
   docker exec infinity-emb ps aux | grep infinity_emb
   # 应看到 --url-prefix /v1
   ```

2. 确认 RAGFlow 中模型 Base URL 为 `http://infinity-emb:8080`（**不要**手动加 `/v1`，RAGFlow 会自动加）。

3. 确认 Provider 选择 `OpenAI-API-Compatible`。

4. API Key 可填任意非空字符串（如 `sk-infinity`）。

5. 手动验证 endpoint 是否存在：

   ```bash
   curl -s -o /dev/null -w "%{http_code}\n" \
     -X POST http://localhost:8080/v1/embeddings \
     -H "Content-Type: application/json" \
     -d '{"model":"BAAI/bge-m3","input":["hello"]}'
   # 预期：200
   ```

6. 修改后必须重建并重启 `infinity-emb`：

   ```bash
   cd /path/to/ragflow
   docker build -f Dockerfile.infinity-emb -t infiniflow/infinity-emb:gpu .
   cd docker
   docker compose -f docker-compose-dgx-arm64.yml stop infinity-emb
   docker compose -f docker-compose-dgx-arm64.yml rm -f infinity-emb
   docker compose -f docker-compose-dgx-arm64.yml --profile elasticsearch up -d infinity-emb
   ```

### 12.10 模型/依赖下载失败

- 检查网络连通性。
- 国内环境设置 `NEED_MIRROR=1` 与 `HF_ENDPOINT=https://hf-mirror.com`。
- 离线环境需提前将 `ragflow_deps` 镜像与模型缓存准备好。

### 12.11 UI 版本号显示为 commit hash 而非 `v0.26.1`

RAGFlow 主镜像的版本号由构建时的 Git tag 决定。若直接基于源码构建且未打 tag，UI 会显示类似 `ede46e0bb` 的 commit hash。修复方法：

```bash
cd /path/to/ragflow

# 确认当前代码对应 v0.26.1
git log --oneline -1
# 例如输出：ede46e0bb Merge pull request #.... from ...

# 在本地打 tag（若官方 tag 缺失）
git tag v0.26.1 ede46e0bb

# 重新构建主镜像
docker build -f Dockerfile -t infiniflow/ragflow:v0.26.1 .
```

重建后，在 `.env` 中改用 `RAGFLOW_IMAGE=infiniflow/ragflow:v0.26.1` 并重启容器，UI 即可正确显示 `v0.26.1`。

---

## 13. 附录

### 13.1 Go 后端构建（可选）

若希望使用 `API_PROXY_SCHEME=go` 或 `hybrid`，需在 ARM64 宿主机上构建 Go 二进制并打包到镜像中。

#### 13.1.1 宿主机准备

```bash
# Ubuntu/Debian ARM64
sudo apt-get update
sudo apt-get install -y cmake g++ libpcre2-dev libsimde-dev golang-go

# 确认 Go 版本
go version
# 要求 >= 1.26.4（由 go.mod 指定）

# 升级 CMake 到 >= 4.0（如系统版本不足）
pip3 install --upgrade 'cmake>=4.0,<5'
```

#### 13.1.2 构建 Go 二进制

```bash
cd /path/to/ragflow

# 该脚本会自动下载 office_oxide native-linux-aarch64、构建 C++ tokenizer、编译 Go 服务
./build.sh --all

# 产物位于 bin/
ls -la bin/
# ragflow_server, admin_server, ingestion_server, ragflow-cli
```

#### 13.1.3 打包到镜像

在主 `Dockerfile` 的 `production` stage 中增加 COPY：

```dockerfile
COPY bin/ragflow_server bin/admin_server bin/ingestion_server bin/ragflow-cli /ragflow/bin/
RUN chmod +x /ragflow/bin/*
```

#### 13.1.4 启用 Go 后端

```bash
# .env
API_PROXY_SCHEME=go
COMPOSE_PROFILES=elasticsearch,cpu,ragflow-go
```

> **注意**：Go 后端目前仍在快速迭代中，生产环境建议先使用 `python` 模式。

### 13.2 关键命令速查

```bash
# 应用 ARM64 补丁
cd /path/to/ragflow
./patch_dgx_arm64.sh

# 构建依赖镜像
cd ragflow_deps
uv run download_deps.py
docker build -f Dockerfile -t infiniflow/ragflow_deps:latest .

# 构建 ARM64 主镜像
cd ..
docker buildx create --use --name ragflow-builder --driver docker-container || true
docker buildx build --platform linux/arm64 --build-arg NEED_MIRROR=0 \
  -f Dockerfile -t <your-registry>/ragflow:v0.26.1-arm64 --push .

# 构建 infinity-emb GPU 镜像
cd /opt/ragflow-prod
docker build -f Dockerfile.infinity-emb -t ragflow/infinity-emb:v0.0.66-cu124-arm64 .

# 启动基础设施
docker compose -f docker-compose-base.yml -f docker-compose.yml \
  --profile elasticsearch --profile cpu up -d mysql es01 minio redis

# 启动 RAGFlow 主容器
docker compose -f docker-compose-base.yml -f docker-compose.yml \
  --profile elasticsearch --profile cpu up -d ragflow-cpu

# 启动 infinity-emb（如使用 compose profile）
docker compose -f docker-compose-base.yml -f docker-compose.yml \
  --profile elasticsearch --profile cpu --profile infinity-emb up -d

# 查看日志
docker compose logs -f ragflow-cpu
docker logs -f infinity-emb

# 健康检查
curl -s http://localhost:9380/api/v1/system/healthz
curl -s http://localhost:8080/health
curl -u elastic:<password> http://localhost:1200/_cluster/health

# 停止并清理（保留卷）
docker compose -f docker-compose-base.yml -f docker-compose.yml \
  --profile elasticsearch --profile cpu down

# 完全清理（含卷，生产慎用）
docker compose -f docker-compose-base.yml -f docker-compose.yml \
  --profile elasticsearch --profile cpu down -v
```

### 13.3 参考文档

- RAGFlow 官方仓库：https://github.com/infiniflow/ragflow
- RAGFlow 构建镜像文档：`docs/develop/build_docker_image.mdx`
- NVIDIA Container Toolkit 文档：https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/
- Elasticsearch Docker 部署：https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
- Infinity-emb 文档：https://github.com/michaelfeil/infinity_emb
- PyTorch ARM64 wheel：https://download.pytorch.org/whl/

---

> **免责声明**：本指南基于 RAGFlow v0.26.1 代码分析编写。由于上游版本迭代较快，部分路径、版本号、依赖关系可能在后续版本中发生变化。生产部署前，请在非生产环境完成全量功能验证与压力测试。
