# RAGFlow 生产环境运维手册

> **适用对象**：基于 Docker Compose 部署的 RAGFlow 生产环境，包括 DGX Spark ARM64 服务器  
> **目标**：提供日常运维、故障排查、备份恢复、自动化运维的完整指南  
> **最后更新**：2026-06-26

---

## 1. 运维概述

RAGFlow 生产环境通常由以下容器组成：

| 容器 | 角色 | 关键依赖 |
|------|------|----------|
| `ragflow` | 主 API、任务调度、文档解析 | MySQL、ES、Redis、MinIO |
| `es01` | 向量 + 全文检索 | 持久化卷 |
| `mysql` | 元数据数据库 | 持久化卷 |
| `minio` | 对象存储（文档、图片） | 持久化卷 |
| `redis` / `valkey` | 缓存、任务队列 | 持久化卷 |
| `infinity-emb` | GPU embedding/reranker | GPU |
| `mineru-api` | GPU PDF 解析（可选） | GPU |
| `mineru-vllm-server` | MinerU 独立 vLLM 后端（可选） | GPU |
| `nginx` | Web/API 代理 | - |

---

## 2. 日常巡检清单

### 2.1 每日检查

```bash
# 1. 检查所有容器状态
docker compose ps

# 2. 检查容器资源使用
docker stats --no-stream

# 3. 检查 GPU 状态
nvidia-smi

# 4. 检查关键服务健康
curl -s -o /dev/null -w "%{http_code}" http://localhost:9380/
curl -s -o /dev/null -w "%{http_code}" http://localhost:9200/_cluster/health
curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/openapi.json
```

### 2.2 每周检查

1. 查看日志中是否有 ERROR/FATAL：
   ```bash
   docker logs ragflow --since 7d | grep -iE "error|fatal|exception" | tail -50
   ```
2. 检查磁盘空间：
   ```bash
   df -h
   docker system df
   ```
3. 检查 MinIO 存储增长趋势。
4. 检查 Elasticsearch 集群健康：
   ```bash
   curl http://localhost:9200/_cluster/health?pretty
   ```

### 2.3 每月检查

1. 评估资源使用趋势，决定是否扩容。
2. 清理过期日志和临时文件。
3. 检查安全补丁和镜像更新。
4. 验证备份可用性（做一次恢复演练）。

---

## 3. 服务启停流程

### 3.1 启动全部服务

```bash
cd docker

# DGX ARM64
docker compose -f docker-compose-dgx-arm64.yml up -d

# 可选：启动 MinerU GPU 解析服务（使用已验证的 tools/mineru-gpu/ 部署）
cd ../tools/mineru-gpu
docker compose up -d
```

### 3.2 停止全部服务

```bash
# 停止 MinerU（如启用）
cd /path/to/ragflow/tools/mineru-gpu
docker compose down

# 停止主服务
cd /path/to/ragflow/docker
docker compose -f docker-compose-dgx-arm64.yml down
```

### 3.3 重启单个服务

```bash
# 重启 RAGFlow
docker restart ragflow

# 重启 MinerU API
docker restart mineru-api

# 重启 infinity-emb
docker restart infinity-emb
```

### 3.4 滚动重启

避免全部服务同时重启导致中断：

```bash
# 先重启无状态服务
docker restart nginx

# 再重启 RAGFlow
docker restart ragflow

# 等待 RAGFlow 健康后再重启依赖
docker restart infinity-emb
docker restart mineru-api
```

---

## 4. 日志查看

### 4.1 实时日志

```bash
docker logs -f ragflow
docker logs -f mineru-api
docker logs -f mineru-vllm-server
docker logs -f infinity-emb
docker logs -f es01
```

### 4.2 按时间过滤

```bash
# 最近 10 分钟
docker logs --since 10m ragflow

# 最近 100 行
docker logs --tail 100 ragflow
```

### 4.3 关键字搜索

```bash
# MinerU 相关
docker logs ragflow | grep -i mineru

# 错误信息
docker logs ragflow | grep -iE "error|exception|fatal"

# 任务执行
docker logs ragflow | grep "handle_task"
```

### 4.4 日志持久化建议

默认 Docker 日志存储在 `/var/lib/docker/containers/`，建议配置日志驱动限制：

```yaml
services:
  ragflow:
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "5"
```

---

## 5. 备份与恢复

### 5.1 需要备份的数据

| 数据 | 位置 | 备份方式 |
|------|------|----------|
| MySQL 数据库 | Docker volume `docker_mysql_data` | `mysqldump` |
| Elasticsearch 索引 | Docker volume `docker_esdata` | 快照 |
| MinIO 对象 | Docker volume `docker_minio_data` | `mc mirror` |
| 模型文件 | `/home/d0ngfq/models/` | rsync |
| 配置文件 | `docker/.env`, `docker-compose*.yml` | git |

### 5.2 MySQL 备份

```bash
# 备份
docker exec docker-mysql-1 mysqldump -u root -p rag_flow > /backup/ragflow_$(date +%Y%m%d).sql

# 恢复
docker exec -i docker-mysql-1 mysql -u root -p rag_flow < /backup/ragflow_20260101.sql
```

### 5.3 Elasticsearch 快照

配置快照仓库并创建快照：

```bash
# 注册仓库
curl -X PUT "localhost:9200/_snapshot/ragflow_backup" -H 'Content-Type: application/json' -d'
{
  "type": "fs",
  "settings": {
    "location": "/usr/share/elasticsearch/backup"
  }
}'

# 创建快照
curl -X PUT "localhost:9200/_snapshot/ragflow_backup/snapshot_$(date +%Y%m%d)?wait_for_completion=true"
```

### 5.4 MinIO 备份

```bash
# 使用 mc 镜像到本地或远端
mc mirror local/ragflow /backup/minio/ragflow
```

### 5.5 整机备份脚本示例

```bash
#!/bin/bash
BACKUP_DIR=/backup/ragflow/$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

# MySQL
docker exec docker-mysql-1 mysqldump -u root -p${MYSQL_PASSWORD} rag_flow > $BACKUP_DIR/mysql.sql

# .env 和 compose 文件
cp docker/.env $BACKUP_DIR/
cp docker/docker-compose*.yml $BACKUP_DIR/

# 模型文件
rsync -av /home/d0ngfq/models/ $BACKUP_DIR/models/

echo "Backup completed: $BACKUP_DIR"
```

---

## 6. 升级流程

### 6.1 升级前准备

1. 阅读新版本 release notes。
2. 完整备份数据。
3. 在测试环境验证升级。

### 6.2 标准升级步骤

```bash
# 1. 停止服务
cd docker
docker compose down

# 2. 拉取最新代码
cd ..
git pull origin main

# 3. 重新构建镜像
docker build -f Dockerfile -t infiniflow/ragflow:latest .

# MinerU GPU 解析服务（使用已验证的 tools/mineru-gpu/ 路径）
cd tools/mineru-gpu
docker build -t mineru:latest -f Dockerfile .
cd ../..

docker build -f Dockerfile.infinity-emb -t infiniflow/infinity-emb:latest .

# 4. 数据库迁移（如有）
# 参考 release notes 执行 migration

# 5. 启动服务
cd docker
docker compose up -d
```

### 6.3 回滚

如果升级失败：

```bash
# 停止服务
docker compose down

# 恢复旧镜像
docker tag infiniflow/ragflow:backup infiniflow/ragflow:latest

# 恢复数据库
docker exec -i docker-mysql-1 mysql -u root -p rag_flow < /backup/ragflow_xxx.sql

# 启动
docker compose up -d
```

---

## 7. 故障排查手册

### 7.1 服务无法启动

```bash
# 查看详细错误
docker compose logs <service-name>

# 检查端口冲突
ss -tlnp | grep <port>

# 检查磁盘空间
df -h
```

### 7.2 RAGFlow 页面无法访问

1. 检查 nginx 是否启动。
2. 检查 RAGFlow 容器是否健康。
3. 检查防火墙：`ufw status` / `iptables -L`。
4. 查看 RAGFlow 日志是否有启动错误。

### 7.3 PDF 解析失败

1. 检查 MinerU 服务是否健康。
2. 检查 RAGFlow 日志中的 MinerU 错误。
3. 检查 GPU 显存是否足够。
4. 参考 `tools/mineru-gpu/README.md` 常见问题。

### 7.4 检索质量下降

1. 检查 embedding 服务是否可用。
2. 检查 ES 索引健康状态。
3. 检查知识库配置（chunk size、parser、reranker）。
4. 查看搜索结果评分。

### 7.5 性能下降

1. 检查 CPU、内存、GPU 使用率。
2. 检查 ES 查询延迟。
3. 检查是否有大任务阻塞队列。
4. 查看 RAGFlow 任务执行日志。

---

## 8. 自动化运维建议

### 8.1 使用 systemd 管理开机启动

创建 `/etc/systemd/system/ragflow.service`：

```ini
[Unit]
Description=RAGFlow Docker Compose
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/d0ngfq/project/ragflow/docker
ExecStart=/usr/bin/docker compose -f docker-compose-dgx-arm64.yml up -d
ExecStop=/usr/bin/docker compose -f docker-compose-dgx-arm64.yml down

[Install]
WantedBy=multi-user.target
```

启用：

```bash
sudo systemctl enable ragflow.service
sudo systemctl start ragflow.service
```

### 8.2 定时备份（cron）

```bash
# 每天凌晨 2 点备份
0 2 * * * /home/d0ngfq/project/ragflow/scripts/backup.sh >> /var/log/ragflow_backup.log 2>&1

# 每周清理旧备份（保留 30 天）
0 3 * * 0 find /backup/ragflow -maxdepth 1 -type d -mtime +30 -exec rm -rf {} \;
```

### 8.3 健康检查自动化

创建 `/home/d0ngfq/project/ragflow/scripts/health_check.sh`：

```bash
#!/bin/bash
SERVICES=("http://localhost:9380/" "http://localhost:9200/_cluster/health" "http://localhost:8000/openapi.json")
for url in "${SERVICES[@]}"; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "$url")
  if [ "$code" != "200" ]; then
    echo "$(date) ALERT: $url returned $code" >> /var/log/ragflow_health.log
    # 可接入邮件/企业微信/钉钉告警
  fi
done
```

定时执行：

```bash
*/5 * * * * /home/d0ngfq/project/ragflow/scripts/health_check.sh
```

### 8.4 日志聚合

推荐使用以下方案之一：

- **Prometheus + Grafana**：监控容器资源、GPU、服务指标
- **ELK / Loki**：集中收集和分析日志
- **Promtail + Grafana Loki**：轻量日志聚合

### 8.5 CI/CD 建议

1. 使用 Git 管理 `docker/.env`（敏感信息用 `.env.example`）。
2. 使用 GitHub Actions / GitLab CI 构建镜像。
3. 测试环境验证后再部署生产。
4. 镜像版本化，保留可回滚标签。

---

## 9. 安全建议

1. **不要直接暴露 RAGFlow 到公网**：使用 Nginx + HTTPS + 认证。
2. **定期更新基础镜像**：修复安全漏洞。
3. **限制容器权限**：使用非 root 用户运行容器（如支持）。
4. **数据库强密码**：`.env` 中的密码使用随机生成。
5. **备份加密**：敏感备份数据加密存储。

---

## 10. 参考

- [RAGFlow DGX ARM64 部署指南](./deploy_dgx_arm64.md)
- [RAGFlow + MinerU GPU 部署指南](./deploy_mineru_gpu_dgx_arm64.md)
- [DGX Spark 统一内存架构优化方案](./dgx_spark_optimization_guide.md)
- `tools/mineru-gpu/README.md`
