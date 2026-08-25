# 灵枢（Lingshu）Docker 安装与部署指南

> 适用环境：Docker Desktop（Windows）或 Docker Engine + Compose（Linux）
> 最后更新：2026-08-25

本项目默认通过 Docker Compose 启动 API、PostgreSQL、Qdrant、IDA MCP 与 AntSword。API 容器可通过 Docker Socket 为每个会话创建独立的 Kali 动态工作区。

## 1. 前置条件

- Docker Desktop（Windows）或 Docker Engine 24+（Linux）；确认 `docker compose version` 可用。
- 至少预留 50 GB 磁盘空间：Kali 工作区基础/派生镜像体积较大。
- 已准备 LLM、Embedding 等服务的访问配置。
- Linux 上，启动用户必须具有 Docker Socket 访问权限：

  ```bash
  sudo usermod -aG docker $USER
  newgrp docker
  ```

## 2. 配置根目录 `.env`

从模板创建根目录 `.env`，并填写密钥、模型地址和数据库密码：

```bash
cp .env.example .env
```

**API 容器读取的是项目根目录 `.env`，不是 `Docker/.env`。** Compose 既以它进行变量替换，也通过 `env_file: ../.env` 将变量注入 API 与初始化容器。`.env` 已被 `.dockerignore` 排除，不会写入镜像。

至少确认以下变量：

```dotenv
# LLM / Embedding / 数据库变量按 .env.example 填写
LINGSHU_POSTGRES_PASSWORD=replace-with-a-strong-password

# WebUI 宿主机端口；默认 8080，如被占用可改为 8081
LINGSHU_APP_PORT=8080

# Docker 动态工作区基础镜像
LINGSHU_WORKSPACE_BASE_IMAGE=gitciu8080/lingshu-kali:latest
```

### Windows（Docker Desktop）工作区路径

容器内路径和 Docker Desktop 看到的 Windows 路径必须分别配置。以下示例假定仓库位于 `G:\比赛\Lingshu\Lingshu`：

```dotenv
LINGSHU_WORKSPACE_ROOT=/workspace/WorkDir/v2
LINGSHU_WORKSPACE_DOCKER_HOST_ROOT=G:/比赛/Lingshu/Lingshu/WorkDir/v2
LINGSHU_AGENT_SERVICE_WORKSPACE_ROOT=/workspace/WorkDir/v2/agent-service
LINGSHU_AGENT_SERVICE_DOCKER_HOST_ROOT=G:/比赛/Lingshu/Lingshu/WorkDir/v2/agent-service
```

请在 Docker Desktop 的文件共享设置中允许该盘符/目录访问。项目会在首次创建会话时生成相应目录。

### Linux 工作区路径

在 Linux 上，两个“宿主机”变量通常与对应容器路径相同：

```dotenv
LINGSHU_WORKSPACE_ROOT=/srv/lingshu/WorkDir/v2
LINGSHU_WORKSPACE_DOCKER_HOST_ROOT=/srv/lingshu/WorkDir/v2
LINGSHU_AGENT_SERVICE_WORKSPACE_ROOT=/srv/lingshu/WorkDir/v2/agent-service
LINGSHU_AGENT_SERVICE_DOCKER_HOST_ROOT=/srv/lingshu/WorkDir/v2/agent-service
```

确保这些目录存在，且 Docker 与 API 容器用户（UID/GID `10001`）可读写。

## 3. 首次启动

在项目根目录执行：

```bash
docker compose --env-file .env -f Docker/docker-compose.yml up -d
```

该命令会自动完成以下工作：

1. 拉取 `.env` 中 `LINGSHU_API_IMAGE` 指定的 API 镜像（默认 `gitciu8080/lingshu-api:latest`）；
2. 启动 PostgreSQL、Qdrant、IDA MCP、AntSword；
3. 运行 `scripts/knowledge_cli.py build`，将 `.RAG` 文档增量写入 Qdrant；
4. 运行 `scripts/build_workspace_image.py`，构建/检查 AgentScope 派生工作区镜像；
5. 初始化成功后启动 API。

首次知识库构建会向 `.env` 中配置的 Embedding 服务发送知识库文本，处理时间取决于文档数量与服务速度。后续启动只会处理新增或变更的文档。向量数据库和应用数据均存放在 Docker 命名卷中。

检查状态：

```bash
docker compose --env-file .env -f Docker/docker-compose.yml ps
docker compose --env-file .env -f Docker/docker-compose.yml logs -f app
```

WebUI 地址为 `http://127.0.0.1:${LINGSHU_APP_PORT}`；例如端口设为 `8081` 时访问 `http://127.0.0.1:8081`。PostgreSQL、Qdrant、IDA MCP 与 AntSword 默认只在 Compose 内网暴露。

## 4. 验证

```bash
# API / WebUI
curl -f http://127.0.0.1:8080/

# 所有容器状态
docker compose --env-file .env -f Docker/docker-compose.yml ps

# API 容器是否可创建动态 Docker 工作区
docker compose --env-file .env -f Docker/docker-compose.yml exec app docker info
```

若将 `LINGSHU_APP_PORT` 设为其他端口，请相应替换上述 `8080`。

知识库状态可通过初始化任务日志确认：

```bash
docker compose --env-file .env -f Docker/docker-compose.yml logs knowledge-init
```

成功时会显示 Qdrant 后端、Embedding 模型以及知识库/记忆库文档数量。

## 5. 日常运维

```bash
# 启动或更新服务（知识库与派生镜像均做增量检查）
docker compose --env-file .env -f Docker/docker-compose.yml up -d

# 停止服务，保留所有数据卷
docker compose --env-file .env -f Docker/docker-compose.yml down

# 查看应用日志
docker compose --env-file .env -f Docker/docker-compose.yml logs -f app

# 修改根目录 .env 后，重新创建 API 容器
docker compose --env-file .env -f Docker/docker-compose.yml up -d --force-recreate app
```

`/var/run/docker.sock` 会挂载进 API 容器，以支持动态工作区生命周期管理。它等同于授予容器管理 Docker 的权限；仅应在受信任的单机或受控环境中部署，且 WebUI 默认只绑定 `127.0.0.1`。

## 6. 常见问题

### 端口被占用

将根目录 `.env` 中的 `LINGSHU_APP_PORT` 改为空闲端口，例如 `8081`，然后重新创建 API：

```bash
docker compose --env-file .env -f Docker/docker-compose.yml up -d --force-recreate app
```

### API 一直处于 Created 状态

先检查两个一次性初始化任务：

```bash
docker compose --env-file .env -f Docker/docker-compose.yml ps -a
docker compose --env-file .env -f Docker/docker-compose.yml logs knowledge-init workspace-init
```

两者都应以退出码 `0` 结束。修复 `.env`、Embedding 服务或 Docker Socket 权限后，再运行：

```bash
docker compose --env-file .env -f Docker/docker-compose.yml up -d app
```

### 动态工作区创建失败

```bash
docker compose --env-file .env -f Docker/docker-compose.yml exec app docker info
docker compose --env-file .env -f Docker/docker-compose.yml logs workspace-init
```

- Linux：确认 API 容器可访问 `/var/run/docker.sock`，且工作区目录权限正确。
- Windows：确认 `LINGSHU_WORKSPACE_DOCKER_HOST_ROOT` 是 Docker Desktop 可访问的绝对路径，且盘符已共享。

### 需要重建派生工作区镜像

```bash
docker compose --env-file .env -f Docker/docker-compose.yml run --rm workspace-init
```

修改 `LINGSHU_WORKSPACE_BASE_IMAGE` 或 `LINGSHU_WORKSPACE_DOCKER_EXTRA_PIP` 后，派生镜像标签会变化并自动重建。
