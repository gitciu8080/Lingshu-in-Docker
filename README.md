# 灵枢（Lingshu）Docker 部署

灵枢是用于**授权安全测试**的多智能体系统。本仓库是独立的 Docker 部署包：应用代码、WebUI 与内置 `.RAG` 原始知识库均来自已发布的镜像，不需要 Python、Conda 或本地源码构建。

## 快速启动

1. 安装 Docker Desktop（Windows）或 Docker Engine 与 Compose（Linux），并确认：

   ```powershell
   docker compose version
   ```

2. 在根目录创建并配置 `.env`：

   ```powershell
   Copy-Item .env.example .env
   ```

   至少填写 LLM、Embedding、PostgreSQL 密码和工作区路径。当前 Windows 部署默认使用：

   ```dotenv
   LINGSHU_API_IMAGE=gitciu8080/lingshu-api:latest
   LINGSHU_APP_PORT=8081
   LINGSHU_WORKSPACE_ROOT=/workspace/WorkDir/v2
   LINGSHU_WORKSPACE_DOCKER_HOST_ROOT=C:/Users/gitci/Desktop/lingshu-Docker/WorkDir/v2
   LINGSHU_AGENT_SERVICE_WORKSPACE_ROOT=/workspace/WorkDir/v2/agent-service
   LINGSHU_AGENT_SERVICE_DOCKER_HOST_ROOT=C:/Users/gitci/Desktop/lingshu-Docker/WorkDir/v2/agent-service
   ```

3. 拉取镜像并启动：

   ```powershell
   docker compose --env-file .env -f Docker/docker-compose.yml pull
   docker compose --env-file .env -f Docker/docker-compose.yml up -d
   ```

4. 打开 WebUI：`http://127.0.0.1:8081`。如果修改了 `LINGSHU_APP_PORT`，请使用对应端口。

首次启动会自动：

1. 启动 PostgreSQL、Qdrant、IDA MCP 与 AntSword；
2. 将镜像中的 `.RAG` 文档向量化并写入 Qdrant；
3. 构建或检查 AgentScope 动态工作区派生镜像；
4. 在初始化成功后启动 API。

首次向量化会将知识库文本发送至 `.env` 中配置的 Embedding 服务，耗时取决于服务速度。请确保已获准向该服务发送这些内容。

## 验证

```powershell
# 服务状态
docker compose --env-file .env -f Docker/docker-compose.yml ps

# API / WebUI
curl.exe -f http://127.0.0.1:8081/

# API 是否可管理动态 Docker 工作区
docker compose --env-file .env -f Docker/docker-compose.yml exec app docker info

# 查看知识库初始化结果
docker compose --env-file .env -f Docker/docker-compose.yml logs knowledge-init
```

`knowledge-init` 与 `workspace-init` 均应以退出码 `0` 结束，`app` 应显示为 `healthy`。

## 目录说明

```text
lingshu-Docker/
├── .env                         # 本机密钥与部署配置；不得提交
├── .env.example                 # 配置模板
├── Docker/
│   ├── docker-compose.yml        # 服务编排
│   ├── postgres-init/            # PostgreSQL 初始化 SQL
│   └── README.md                 # 编排细节
├── WorkDir/v2/                   # 动态工作区宿主机目录
└── install_guide.md              # 完整配置与排障指南
```

## 数据与持久化

- 原始 `.RAG` 知识文件包含在 API 镜像中。
- 向量索引保存在 Qdrant Docker 命名卷中。
- 任务记忆、长期记忆、报告、PostgreSQL 与 MCP 数据也使用 Docker 命名卷。
- `docker compose down` 只停止容器，默认保留以上数据。

将本部署包复制到另一台机器时，原始知识文件会随 API 镜像获取；首次启动会自动重建该机器的 Qdrant 向量索引。

## 日常操作

```powershell
# 启动服务
docker compose --env-file .env -f Docker/docker-compose.yml up -d

# 查看 API 日志
docker compose --env-file .env -f Docker/docker-compose.yml logs -f app

# 停止服务并保留数据卷
docker compose --env-file .env -f Docker/docker-compose.yml down

# 修改 .env 后重新创建 API 容器
docker compose --env-file .env -f Docker/docker-compose.yml up -d --force-recreate app
```

## 安全注意事项

- 仅用于已获得明确授权的安全测试。
- `.env` 含密钥与密码；已在 Git 中暂存时，请执行 `git restore --staged .env`，并确保 `.gitignore` 包含 `.env`。
- API 容器挂载 Docker Socket，以创建每会话工作区容器。这相当于容器拥有 Docker 管理权限，仅应在受信任环境部署。
- WebUI 默认只绑定到 `127.0.0.1`，不要在未配置访问控制的情况下直接公开端口。

详细的环境变量、Linux 部署与故障排查请见 [install_guide.md](install_guide.md)；Compose 服务说明请见 [Docker/README.md](Docker/README.md)。
