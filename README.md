# 本机全局共享 PostgreSQL 18 / Redis / Temporal

这个目录用于启动一组本机长期运行的基础容器，当前包含：

- PostgreSQL 18
- Redis 8
- Temporal CLI dev server（含 UI）

它们的定位是给多个本地项目共用，而不是每个项目各自启动一套。

## 文件说明

- `docker-compose.yaml`：定义全局共享的 `postgres18`、`redis` 和 `temporal` 服务。
- `.env`：本机实际使用的启动参数。
- `.env.example`：环境变量模板，新机器可以先复制它生成 `.env`。
- `consumer-compose.example.yaml`：其他 Docker 项目接入这套共享基础设施的最小示例。

## 当前设计

- PostgreSQL 通过 `localhost:${POSTGRES_PORT}` 提供给宿主机项目使用。
- Redis 通过 `localhost:${REDIS_PORT}` 提供给宿主机项目使用。
- Temporal 通过 `localhost:${TEMPORAL_GRPC_PORT}` 提供 gRPC，通过
  `localhost:${TEMPORAL_UI_PORT}` 提供本地 UI。
- 同时保留 `local-infra` 网络，方便其他 Docker 容器加入同一个网络后访问
  `postgres18:5432`、`redis:6379` 和 `temporal:7233`。
- PostgreSQL 不预置业务数据库，适合多个项目共用一个实例后自行建库。
- PostgreSQL 和 Redis 都使用 Docker 命名卷持久化数据。
- Temporal 使用 SQLite 命名卷持久化本地 Workflow、Schedule 和 History。
- 多个项目共享 Temporal 实例，但必须分别使用独立 Namespace、Task Queue 和 Worker。
- PostgreSQL 18 按官方推荐方式挂载到 `/var/lib/postgresql`。
- Redis 使用 `appendonly yes`，更适合本机共享开发场景，重启后不容易丢最近写入的数据。

## 准备 `.env`

第一次使用时，可以先复制模板：

```bash
cp .env.example .env
```

默认模板内容如下：

```dotenv
POSTGRES_PORT=5432
POSTGRES_USER=postgres
POSTGRES_PASSWORD=change-me
POSTGRES_DB=postgres
REDIS_PORT=6379
REDIS_PASSWORD=
TEMPORAL_GRPC_PORT=7233
TEMPORAL_UI_PORT=8233
TZ=Asia/Shanghai
```

建议你至少修改 `POSTGRES_PASSWORD`。如果这台机器不是你个人独占使用，建议把它当成必须项。
`POSTGRES_DB` 用于首次初始化时创建默认数据库；建议保留为 `postgres`，业务库按项目单独创建。
`REDIS_PASSWORD` 默认留空，表示本机开发环境下不启用认证；如果你要启用认证，填入密码后重建 Redis 容器即可。

## 启动

启动全部服务：

```bash
docker compose up -d
```

如果只单独启动 PostgreSQL：

```bash
docker compose up -d postgres18
```

如果只单独启动 Redis：

```bash
docker compose up -d redis
```

如果只单独启动 Temporal：

```bash
docker compose up -d temporal
```

查看状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f postgres18
docker compose logs -f redis
docker compose logs -f temporal
```

停止服务：

```bash
docker compose down
```

注意：`docker compose down` 不会删除命名卷，因此数据库和缓存数据会保留。

## 宿主机项目如何连接

### PostgreSQL

```text
Host: localhost
Port: ${POSTGRES_PORT}
User: ${POSTGRES_USER}
Password: ${POSTGRES_PASSWORD}
```

默认连接串：

```text
postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@localhost:${POSTGRES_PORT}/${POSTGRES_DB}
```

这里的默认数据库来自 `POSTGRES_DB`，仅在首次初始化数据目录时创建。

### Redis

```text
Host: localhost
Port: ${REDIS_PORT}
```

默认连接串：

```text
redis://localhost:${REDIS_PORT}
```

如果设置了 `REDIS_PASSWORD`，则连接串应写成：

```text
redis://:${REDIS_PASSWORD}@localhost:${REDIS_PORT}
```

### Temporal

宿主机项目使用：

```text
Address: 127.0.0.1:${TEMPORAL_GRPC_PORT}
Protocol: gRPC
UI: http://127.0.0.1:${TEMPORAL_UI_PORT}
```

每个项目首次接入时创建独立 Namespace，例如：

```bash
docker compose exec temporal temporal operator namespace create \
  --address 127.0.0.1:7233 \
  --namespace project-a-local \
  --retention 72h
```

Namespace 建议使用 `<project>-local` 命名。项目还应使用带项目前缀的 Task Queue，并只启动
轮询本项目 Namespace 的 Worker。共享实例适合日常本地开发；会清空状态、修改集群级配置或
验证特定 Temporal Server 版本的自动化测试，应启动独立临时环境。

## Docker 项目如何连接

如果你的应用也运行在 Docker 里，并显式加入了同一个 `local-infra` 网络，则可以直接使用容器名访问：

```text
Postgres host: postgres18
Postgres port: 5432
Redis host: redis
Redis port: 6379
Temporal gRPC address: temporal:7233
```

其他 Compose 项目需要额外声明这个外部网络，例如：

```yaml
services:
  app:
    networks:
      - local-infra

networks:
  local-infra:
    external: true
```

完整的最小接入示例见 `consumer-compose.example.yaml:1`。这个示例默认读取当前 shell 环境里的 `POSTGRES_USER`、`POSTGRES_PASSWORD`、`APP_POSTGRES_DB` 和 `APP_REDIS_URL`；如果没有提供，就回退到 `postgres`、`change-me`、`project_a` 和 `redis://redis:6379`。

## 为不同项目创建独立数据库

由于这是全局共享的 PostgreSQL，建议每个项目使用各自的数据库，而不是共用默认库。

进入容器：

```bash
docker exec -it postgres18 psql -U ${POSTGRES_USER}
```

创建数据库：

```sql
CREATE DATABASE project_a;
CREATE DATABASE project_b;
```

之后不同项目分别连接各自的数据库，例如：

```text
postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@localhost:${POSTGRES_PORT}/project_a
postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@localhost:${POSTGRES_PORT}/project_b
```

## 数据持久化

当前使用的是 Docker 命名卷：

```yaml
volumes:
  - postgres18_data:/var/lib/postgresql
  - redis_data:/data
  - temporal_data:/home/temporal
```

含义是：

- `postgres18_data` 保存 PostgreSQL 数据。
- `redis_data` 保存 Redis 持久化文件。
- `temporal_data` 保存共享 Temporal 的 SQLite 数据、Workflow、Schedule 和 History。
- 容器删除并重建后，只要不删除这些卷，数据就会保留。

查看卷：

```bash
docker volume ls
```

删除卷会清空数据，请谨慎操作：

```bash
docker volume rm local-containers_postgres18_data
docker volume rm local-containers_redis_data
docker volume rm local-containers_temporal_data
```

## 重要说明

- `POSTGRES_USER` 和 `POSTGRES_PASSWORD` 主要在首次初始化数据目录时生效。
- `POSTGRES_DB` 也是首次初始化数据目录时生效，用来创建默认数据库。
- 如果数据卷已经存在，后续修改这些变量不会自动重建用户或重置密码。
- 如果数据卷已经存在但你改了 `.env` 里的 `POSTGRES_USER` 或 `POSTGRES_DB`，容器本身可能仍能启动，但健康检查和示例连接命令可能会因为新用户或新库并不存在而失败。
- 如果你想重新初始化整个 PostgreSQL，需要先停止容器并删除对应的数据卷。
- `REDIS_PASSWORD` 留空时，Redis 不启用认证，适合本机单人开发环境。
- 如果你启用了 `REDIS_PASSWORD`，其他项目的连接串也要同步改成 `redis://:${REDIS_PASSWORD}@host:port`。
- Redis 修改密码后通常需要重建容器，避免运行参数和文档不一致。
- Temporal 是所有本地项目共享的开发服务；停止容器会中断所有项目，删除数据卷会清空全部
  Namespace 的本地状态。
