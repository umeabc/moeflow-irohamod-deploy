# moeflow-irohamod-deploy

MoeFlow 定制版（iroha10）自部署配置，基于官方 [moeflow-com/moeflow-deploy](https://github.com/moeflow-com/moeflow-deploy) 改造。

> 镜像已发布到 GitHub Container Registry（ghcr.io），由源码仓库 `umeabc/moeflow-irohamod` 构建：
>
> - `ghcr.io/umeabc/moeflow-backend:v1.1.8-iroha10-fix17`
> - `ghcr.io/umeabc/moeflow-frontend:v1.1.7-iroha10-fix17`
>
> `docker compose up` 会自动从 ghcr.io 拉取，无需手动导入。

## 与官方部署的差异（本定制版）

| 项 | 官方 moeflow-deploy | 本仓库 |
|---|---|---|
| Celery 消息队列 | RabbitMQ | **Redis**（`redis:7-alpine`，AOF 持久化，maxmemory 256MB LRU） |
| Celery worker | 2 个（default / output） | **1 个**（同时消费 default+output 队列） |
| gunicorn worker | 4 | **2**（低内存机器友好） |
| MongoDB | 默认 WiredTiger | **`--wiredTigerCacheSizeGB 0.25`**（限制缓存上限） |
| Email 链路 | 有 | **已移除**（注册重构后不再发邮件） |
| 存储 | 仅 LOCAL_STORAGE / OSS | **新增 Cloudflare R2**（`STORAGE_TYPE=R2`，S3 兼容 + 公开桶直读）+ **REMOTE_HTTP**（独立 imgstore 图片存储服务，见下） |

资源占用（2 核 / 2GB 测试机实测）：约 838MB → **381MB**（-55%），可用内存从 49MB 提升至 500MB+。

## 快速开始

0. 安装 [docker](https://docs.docker.com/engine/install/) 和 [docker-compose-plugin](https://docs.docker.com/compose/install/)（docker-compose-plugin v2.27.0 经测试可用）。
1. 复制环境变量模板：
   ```bash
   cp .env.sample .env
   cp .env-backend.sample .env-backend
   ```
2. 编辑 `.env` / `.env-backend`，将 `CHANGE_ME` 替换为实际值（域名、MongoDB 密码、SECRET_KEY、管理员账号等）。
3. 启动（首次会自动从 ghcr.io 拉取镜像）：
   ```bash
   docker compose up -d
   ```
   > 如需指定镜像版本，可在 docker-compose.yml 中修改 `image: ghcr.io/umeabc/moeflow-*` 的 tag 后重新 `docker compose up -d`。

## 服务拓扑

```
frontend (nginx:1.26, /api 反代 backend)  ── 80/443
backend  (Flask + gunicorn -w 2)          ── :5000
celery   (单 worker, 消费 default+output)  ── Redis broker
redis    (消息队列, AOF 持久化)            ── :6379
mongodb  (业务数据, wiredTiger 0.25GB)     ── :27017
```

## Cloudflare R2 存储（可选）

默认 `STORAGE_TYPE=LOCAL_STORAGE`（图片存本地磁盘）。如需改用 Cloudflare R2：

1. 在 Cloudflare R2 创建 bucket，开启 **Public access**（得到 `https://pub-<hash>.r2.dev` 公网域名，或绑自定义域名）
2. 在 R2 管理页创建 **API 令牌**（对象读/写），得到 Access Key / Secret
3. 修改 `.env-backend`：
   ```ini
   STORAGE_TYPE=R2
   STORAGE_DOMAIN=https://pub-<hash>.r2.dev/   # R2 公网域名或自定义域名
   R2_ACCOUNT_ID=<cloudflare account id>
   R2_ACCESS_KEY_ID=<r2 access key>
   R2_SECRET_ACCESS_KEY=<r2 secret>
   R2_BUCKET_NAME=<bucket 名>
   ```
4. `docker compose up -d` 重建 backend/celery（缩略图 cover/safe-check 会自动生成并上传 R2）
5. 存量本地图片迁移：用 `scripts/` 下的迁移脚本（从 `/data/stoarge/files` 上传 R2），或直接复用后端容器内 boto3 逐个 `put_object`

> R2 为 S3 兼容 API（boto3 接入），公开桶 URL 免签名直读，前端展示走 `https://*.r2.dev/xxx.jpg` 外链。

## imgstore 独立图片存储（可选，STORAGE_TYPE=REMOTE_HTTP）

把图片存储落到**另一台独立主机**（轻量 imgstore 服务，Go 单二进制、镜像约 7MB），Moeflow 上传 / 从社交媒体获取的图片直接存到那台主机，外链直读。

1. 在 imgstore 主机部署服务（编排见仓库 `umeabc/moeflow-irohamod-imgstore`）：
   ```bash
   git clone https://github.com/umeabc/moeflow-irohamod-imgstore.git
   cd moeflow-irohamod-imgstore && cp .env.sample .env  # 改 IMGSTORE_API_KEY
   docker compose up -d   # 数据落 ./imgstore-data（绑定映射主机磁盘）
   ```
2. 修改 Moeflow `.env-backend`：
   ```ini
   STORAGE_TYPE=REMOTE_HTTP
   STORAGE_DOMAIN=http://<imgstore主机>:8080/files/   # 外链直读前缀
   REMOTE_HTTP_BASE_URL=http://<imgstore主机>:8080     # 写 API 地址
   REMOTE_HTTP_API_KEY=<与 IMGSTORE_API_KEY 相同>
   ```
3. `docker compose up -d` 重建 backend/celery。之后 dashboard「存储空间」卡片显示 **imgstore 总存储占用**，管理后台新增「imgstore 存储概览」页（服务 URL / 已用 / 剩余 / 总容量）。
4. HTTPS：imgstore 前置 nginx/Caddy 反代后，把 `STORAGE_DOMAIN` 换成 `https://<域名>/files/` 即可（写 API 走 `REMOTE_HTTP_BASE_URL` 不受影响）。

## 目录结构

| 路径 | 说明 |
|---|---|
| `docker-compose.yml` | 服务编排（Redis 版） |
| `.env.sample` / `.env-backend.sample` | 环境变量模板（脱敏，含 Redis 说明） |
| `nginx/templates/` | 前端 nginx 配置模板（/api 反代、/storage、静态资源） |
| `Makefile` | 开发辅助（dev-deps / backend-dev / backend-worker） |

## 数据与备份

- 数据卷：`./mongodb/data/db`（业务数据）、`./redis`（消息队列持久化）、`./storage`（上传文件）
- 升级/迁移：源码仓库构建新镜像并推送 ghcr.io → 部署机修改 `docker-compose.yml` 中的镜像 tag → `docker compose pull && docker compose up -d` 替换 backend/frontend/celery 即可，**保留 mongodb/redis/storage 数据卷**。
