# 灵枢 Docker 部署

本编排运行 Web API、PostgreSQL、Qdrant、IDA MCP 与 AntSword MCP。每个任务的
Kali 工作区由 API 通过 Docker Socket 动态创建，并非 Compose 的固定服务。

## 前置条件

- Linux Docker Engine（推荐 Ubuntu 22.04/24.04）；Windows Docker Desktop 仅建议用于
  API/WebUI 开发，不作为 `NetworkMode=host` 与原始套接字能力的验收环境。
- 已拉取或构建 `kali-full` 工作区镜像，并在 `.env` 中设置
  `LINGSHU_WORKSPACE_BASE_IMAGE`。
- Docker daemon 可被部署用户访问。
- 一个供 API 与 Docker daemon 以相同绝对路径访问的工作区根目录。例如：

```bash
sudo install -d -o 10001 -g 10001 /srv/lingshu/WorkDir/v2
```

该路径约束很重要：API 会把会话目录 bind mount 到动态容器的 `/workspace`；Docker
daemon 看到的源路径必须和 API 传出的路径一致。

## 配置与启动

从项目根目录复制并填写 `.env`，至少设置真实密码、JWT 密钥和模型密钥。增加：

```env
LINGSHU_POSTGRES_PASSWORD=replace-with-a-long-random-password
LINGSHU_WORKSPACE_ROOT=/srv/lingshu/WorkDir/v2
LINGSHU_WORKSPACE_DOCKER_HOST_ROOT=/srv/lingshu/WorkDir/v2
LINGSHU_AGENT_SERVICE_WORKSPACE_ROOT=/srv/lingshu/WorkDir/v2/agent-service
LINGSHU_AGENT_SERVICE_DOCKER_HOST_ROOT=/srv/lingshu/WorkDir/v2/agent-service
LINGSHU_WORKSPACE_BASE_IMAGE=kali-full:latest
```

启动：

```bash
docker compose --env-file .env -f Docker/docker-compose.yml up -d
```

`knowledge-init` 会在 API 启动前运行 `scripts/knowledge_cli.py build`，
`workspace-init` 会运行 `scripts/build_workspace_image.py`。两者均为幂等操作：
后续 `compose up` 只做增量知识同步，并复用已存在的派生镜像。可单独重跑：

```bash
docker compose --env-file .env -f Docker/docker-compose.yml run --rm knowledge-init
docker compose --env-file .env -f Docker/docker-compose.yml run --rm workspace-init
```

WebUI 默认监听 `http://127.0.0.1:8080`，可用 `.env` 的 `LINGSHU_APP_PORT` 改为其他本机端口。PostgreSQL、Qdrant、IDA MCP 与 AntSword
仅在 Compose 内网暴露；需诊断时使用 `docker compose exec`，不要直接公开 MCP 端口。

## 验证与运维

```bash
docker compose --env-file .env -f Docker/docker-compose.yml ps
docker compose --env-file .env -f Docker/docker-compose.yml logs -f app
docker ps --filter label=lingshu.workspace=true
```

创建任务后，应看到带 `lingshu.workspace=true` 标签的独立工作区容器。会话结束或
空闲回收只关闭容器，不会删除工作区文件；用户删除会话时才会将其移动到工作区 trash。

Docker Socket 等价于较高的宿主机权限。生产环境应把 API 放在专用主机或使用仅允许
容器管理操作的 Socket Proxy，并且不要把 API 或 MCP 服务直接暴露到公网。
