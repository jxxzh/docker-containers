# 本机全局共享 PostgreSQL 18 / Redis

这个目录用于启动一组本机长期运行的基础容器，当前包含：

- PostgreSQL 18
- Redis 7

它们的定位是给多个本地项目共用，而不是每个项目各自启动一套。

## 文件说明

- `docker-compose.yaml`：定义全局共享的 `postgres18` 和 `redis` 服务。
- `.env`：本机实际使用的启动参数。
- `.env.example`：环境变量模板，新机器可以先复制它生成 `.env`。
- `consumer-compose.example.yaml`：其他 Docker 项目接入这套共享基础设施的最小示例。

## 当前设计

- PostgreSQL 通过 `localhost:${POSTGRES_PORT}` 提供给宿主机项目使用。
- Redis 通过 `localhost:${REDIS_PORT}` 提供给宿主机项目使用。
- 同时保留 `local-infra` 网络，方便其他 Docker 容器加入同一个网络后访问 `postgres18:5432` 和 `redis:6379`。
- PostgreSQL 不预置业务数据库，适合多个项目共用一个实例后自行建库。
- PostgreSQL 和 Redis 都使用 Docker 命名卷持久化数据。
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
REDIS_PORT=6379
TZ=Asia/Shanghai
```

建议你至少修改 `POSTGRES_PASSWORD`。如果这台机器不是你个人独占使用，建议把它当成必须项。

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

查看状态：

```bash
docker compose ps
```

查看日志：

```bash
docker compose logs -f postgres18
docker compose logs -f redis
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
postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@localhost:${POSTGRES_PORT}/${POSTGRES_USER}
```

因为没有显式设置 `POSTGRES_DB`，官方镜像会在首次初始化时创建与 `POSTGRES_USER` 同名的默认数据库。

### Redis

```text
Host: localhost
Port: ${REDIS_PORT}
```

默认连接串：

```text
redis://localhost:${REDIS_PORT}
```

## Docker 项目如何连接

如果你的应用也运行在 Docker 里，并显式加入了同一个 `local-infra` 网络，则可以直接使用容器名访问：

```text
Postgres host: postgres18
Postgres port: 5432
Redis host: redis
Redis port: 6379
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

完整的最小接入示例见 `consumer-compose.example.yaml:1`。这个示例默认读取当前 shell 环境里的 `POSTGRES_USER`、`POSTGRES_PASSWORD` 和 `APP_POSTGRES_DB`；如果没有提供，就回退到 `postgres`、`change-me` 和 `project_a`。

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
```

含义是：

- `postgres18_data` 保存 PostgreSQL 数据。
- `redis_data` 保存 Redis 持久化文件。
- 容器删除并重建后，只要不删除这些卷，数据就会保留。

查看卷：

```bash
docker volume ls
```

删除卷会清空数据，请谨慎操作：

```bash
docker volume rm local-containers_postgres18_data
docker volume rm local-containers_redis_data
```

## 重要说明

- `POSTGRES_USER` 和 `POSTGRES_PASSWORD` 主要在首次初始化数据目录时生效。
- 如果数据卷已经存在，后续修改这些变量不会自动重建用户或重置密码。
- 如果数据卷已经存在但你改了 `.env` 里的 `POSTGRES_USER`，容器本身可能仍能启动，但健康检查和示例连接命令可能会因为新用户名并不存在而失败。
- 如果你想重新初始化整个 PostgreSQL，需要先停止容器并删除对应的数据卷。
- Redis 这里没有开启密码，默认适合本机单人开发环境；如果你的机器会暴露给其他人使用，建议不要直接开放到不受信任的网络环境。
