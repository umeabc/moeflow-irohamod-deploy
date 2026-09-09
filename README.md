# moeflow-irohamod-deploy

MoeFlow 定制版（iroha10）自部署配置，基于官方 [moeflow-com/moeflow-deploy](https://github.com/moeflow-com/moeflow-deploy) 改造。

> 镜像由源码仓库 `umeabc/moeflow-irohamod` 构建，tag 为：
> - `moeflow-backend:v1.1.8-iroha10-fix4`
> - `moeflow-frontend:v1.1.7-iroha10-fix`
>
> 请先将对应镜像导入部署机（`docker load -i <tar>`），再按本仓库配置启动。

## 与官方部署的差异（本定制版）

| 项 | 官方 moeflow-deploy | 本仓库 |
|---|---|---|
| Celery 消息队列 | RabbitMQ | **Redis**（`redis:7-alpine`，AOF 持久化，maxmemory 256MB LRU） |
| Celery worker | 2 个（default / output） | **1 个**（同时消费 default+output 队列） |
| gunicorn worker | 4 | **2**（低内存机器友好） |
| MongoDB | 默认 WiredTiger | **`--wiredTigerCacheSizeGB 0.25`**（限制缓存上限） |
| Email 链路 | 有 | **已移除**（注册重构后不再发邮件） |

资源占用（2 核 / 2GB 测试机实测）：约 838MB → **381MB**（-55%），可用内存从 49MB 提升至 500MB+。

## 快速开始

0. 安装 [docker](https://docs.docker.com/engine/install/) 和 [docker-compose-plugin](https://docs.docker.com/compose/install/)（docker-compose-plugin v2.27.0 经测试可用）。
1. 准备定制镜像（见上方说明）并 `docker load`。
2. 复制环境变量模板：
   ```bash
   cp .env.sample .env
   cp .env-backend.sample .env-backend
   ```
3. 编辑 `.env` / `.env-backend`，将 `CHANGE_ME` 替换为实际值（域名、MongoDB 密码、SECRET_KEY、管理员账号等）。
4. 启动：
   ```bash
   docker compose up -d
   ```

## 服务拓扑

```
frontend (nginx:1.26, /api 反代 backend)  ── 80/443
backend  (Flask + gunicorn -w 2)          ── :5000
celery   (单 worker, 消费 default+output)  ── Redis broker
redis    (消息队列, AOF 持久化)            ── :6379
mongodb  (业务数据, wiredTiger 0.25GB)     ── :27017
```

## 目录结构

| 路径 | 说明 |
|---|---|
| `docker-compose.yml` | 服务编排（Redis 版） |
| `.env.sample` / `.env-backend.sample` | 环境变量模板（脱敏，含 Redis 说明） |
| `nginx/templates/` | 前端 nginx 配置模板（/api 反代、/storage、静态资源） |
| `Makefile` | 开发辅助（dev-deps / backend-dev / backend-worker） |

## 数据与备份

- 数据卷：`./mongodb/data/db`（业务数据）、`./redis`（消息队列持久化）、`./storage`（上传文件）
- 升级/迁移：`docker save` 新镜像 → 部署机 `docker load` → `docker compose up -d` 替换 backend/frontend/celery 即可，**保留 mongodb/redis/storage 数据卷**。
