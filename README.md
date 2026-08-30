# 灵枢（Lingshu）Docker 部署

> 适用环境：Docker Desktop（Windows）或 Docker Engine + Compose（Linux）

灵枢是用于**授权安全测试**的多智能体系统。本仓库是独立的 Docker 部署包：应用代码、WebUI 与内置 `.RAG` 原始知识库均来自已发布的镜像，不需要 Python、Conda 或本地源码构建。默认通过 Docker Compose 启动 API、PostgreSQL、Qdrant、IDA MCP 与 AntSword，API 容器可通过 Docker Socket 为每个会话创建独立的 Kali 动态工作区。

## 项目介绍

灵枢基于 AgentScope 2.x 构建，面向需要明确授权边界的安全测试与 CTF 分析任务。它的目标不是无约束的"自动攻击"，而是将大模型的开放式推理纳入一条**可审计、可恢复、以证据为依据且受人工闸门约束**的执行轨道。

### 多智能体协作

```text
用户任务
   │
   ▼
Brain（战略规划）
   │
   ▼
Orchestrator（唯一状态决策者）
   ├── Recon / ArtifactTriage：网络侦察与文件初检
   ├── Analysis：漏洞研判与攻击路径分析
   ├── Attack：在授权范围内执行验证
   ├── Reserve：逆向、GDB、IDA 等专项分析
   └── Summary：生成结构化报告
```

只有 Orchestrator 可以推进任务状态；其他专家只返回证据和建议。这种"单调度者 + 专家分工"的拓扑避免多个 Agent 并发改写状态，也让决策顺序和责任主体可追溯。

### 核心能力

| 能力 | 说明 |
|---|---|
| 任务闭环 | 从目标理解、侦察、研判、人工审批、验证到报告生成，采用状态机约束合法阶段迁移。 |
| 授权与最小权限 | 网络命令需显式匹配授权目标；高风险攻击动作经过人工审批；不同专家仅获得职责所需工具。 |
| 证据与审计 | 任务状态、计划、工具调用、审批与终态均记录为 SHA-256 前向哈希审计链，可校验篡改。 |
| 三层记忆 | `.RAG` 安全知识库回答"是什么"，长期经验记忆回答"怎么做"，会话状态保存当前任务事实与上下文。 |
| 隔离执行 | 每个会话可创建独立 Kali Docker 工作区，支持安全工具、逆向分析与 MCP 扩展。 |
| 多模型与报告 | 支持多模型 Provider；任务结束后可生成结构化 Markdown、PDF 和 JSONL 结果。 |

### 安全治理原则

- 明确授权范围优先于模型自主判断；范围外目标与危险命令会被确定性规则拦截。
- 攻击阶段以人工审批和证据要求为闸门，不将模型文本视为成功事实。
- 角色隔离、严格参数校验、动作去重/熔断、上下文治理与审计链共同降低幻觉和越权风险。
- 系统内置约束是纵深防护，不能替代部署者的授权审查与法律责任。

## 前置条件

- Docker Desktop（Windows）或 Docker Engine 24+（Linux）；确认 `docker compose version` 可用。
- 至少预留 50 GB 磁盘空间：Kali 工作区基础/派生镜像体积较大。
- 已准备 LLM、Embedding 等服务的访问配置。
- Linux 上，启动用户必须具有 Docker Socket 访问权限：

  ```bash
  sudo usermod -aG docker $USER
  newgrp docker
  ```

## 快速启动

### 第 1 步：安装 Docker 并确认 Compose

```powershell
docker compose version
```

### 第 2 步：创建 `.env` 并修改必要变量

在根目录从模板创建 `.env`：

```powershell
Copy-Item .env.example .env
```

复制后，将 `.env` 与 `.env.example` 对比，按下面的清单修改。**下表之外的所有变量在模板中已带可用默认值，无需改动。**

|**变量**|**模板值（.env.example）**|**需要改为**|**说明**|
|---|---|---|---|
|`LINGSHU_EMBEDDING_BASE_URL`|`http/`[`https://URL/v1`](https://URL/v1)（占位符）|你的 Embedding 服务 OpenAI 兼容地址，例如 [`http://192.168.120.7:11434/v1`](http://192.168.120.7:11434/v1)|**必改**，知识库向量化依赖它|
|`LINGSHU_POSTGRES_PASSWORD`|`123456`（弱默认值）|强随机密码|**必改**|
|`LINGSHU_DATABASE_URL`|`postgresql://lingshu:`[`replace-with-a-long-random-password@127.0.0.1`](mailto:replace-with-a-long-random-password@127.0.0.1)`:5432/lingshu`|把其中的密码换成与 `LINGSHU_POSTGRES_PASSWORD` 相同的值|**必改**|
|`LINGSHU_SUPERADMIN_PASSWORD`|`gdcp@2026`（示例密码）|仅管理员持有的高强度密码|**必改**，否则超级管理员使用公开示例密码|
|`LINGSHU_REGISTRATION_INVITE_CODE`|`type-key`（示例邀请码）|随机邀请码；留空则关闭公开注册|**必改**，建议留空并只使用 `/superadmin`|
|`LINGSHU_EMBEDDING_API_KEY`|`sk-your-embedding-key`（占位符）|真实 API Key；服务无需认证时可保持占位|**必改**|
|`LINGSHU_EMBEDDING_MODEL`|`nomic-embed-text:latest`|与 Embedding 服务实际部署的模型一致|**必改**|
|`LINGSHU_APP_PORT`|`8083`|端口被占用时改为其他空闲端口|**必改**|



### 第 3 步：拉取镜像并启动

```powershell
docker compose --env-file .env -f Docker/docker-compose.yml pull
docker compose --env-file .env -f Docker/docker-compose.yml up -d
```

### 第 4 步：打开 WebUI

访问 `http://127.0.0.1:${LINGSHU_APP_PORT}`（默认 `http://127.0.0.1:8083`）；如果修改了 `LINGSHU_APP_PORT`，请使用对应端口。

### 首次启动会自动完成

1. 启动 PostgreSQL、Qdrant、IDA MCP 与 AntSword；
2. 运行 `scripts/knowledge_cli.py build`，将镜像中的 `.RAG` 文档向量化并写入 Qdrant；
3. 运行 `scripts/build_workspace_image.py`，构建/检查 AgentScope 动态工作区派生镜像；
4. 在初始化成功后启动 API。

首次向量化会将知识库文本发送至 `.env` 中配置的 Embedding 服务，耗时取决于服务速度。请确保已获准向该服务发送这些内容。后续启动只处理新增或变更的文档，`knowledge-init` 与 `workspace-init` 均为幂等操作。

### 验证

```powershell
# 服务状态（app 应显示为 healthy）
docker compose --env-file .env -f Docker/docker-compose.yml ps

# API / WebUI
curl.exe -f http://127.0.0.1:8083/

# API 是否可管理动态 Docker 工作区
docker compose --env-file .env -f Docker/docker-compose.yml exec app docker info

# 查看知识库初始化结果
docker compose --env-file .env -f Docker/docker-compose.yml logs knowledge-init
```

`knowledge-init` 与 `workspace-init` 均应以退出码 `0` 结束。成功时知识库日志会显示 Qdrant 后端、Embedding 模型以及知识库/记忆库文档数量。创建任务后，可通过 `docker ps --filter label=lingshu.workspace=true` 看到独立的会话工作区容器。

## 完整配置说明

**API 容器读取的是项目根目录 `.env`，不是 `Docker/.env`。** Compose 既以它进行变量替换，也通过 `env_file: ../.env` 将变量注入 API 与初始化容器。`.env` 已被 `.dockerignore` 排除，不会写入镜像。


### 其余可选变量

- `LINGSHU_PDF_FONT_REGULAR` / `LINGSHU_PDF_FONT_BOLD`：PDF 报告字体，未设置时回退到内置中文字体。
- `LINGSHU_WORKSPACE_GATEWAY_PORT`：容器内 MCP 网关节选端口，留空自动分配。
- `LINGSHU_WORKSPACE_DOCKER_EXTRA_PIP`：额外打进派生镜像的 pip 包（逗号分隔，可选）。
- `LINGSHU_IDA_MCP_IMAGE` / `LINGSHU_ANTSWORD_IMAGE`：可固定 Compose 使用的远程 MCP 镜像版本，避免部署时无意升级。
- `LINGSHU_IDA_MCP_URL` / `LINGSHU_ANTSWORD_MCP_URL`：仅当不使用 Compose 内网服务、改为直连宿主 MCP 时设置。

## 日常运维

```powershell
# 启动或更新服务（知识库与派生镜像均做增量检查）
docker compose --env-file .env -f Docker/docker-compose.yml up -d

# 查看 API 日志
docker compose --env-file .env -f Docker/docker-compose.yml logs -f app

# 停止服务并保留数据卷
docker compose --env-file .env -f Docker/docker-compose.yml down

# 修改 .env 后重新创建 API 容器
docker compose --env-file .env -f Docker/docker-compose.yml up -d --force-recreate app
```

## 常见问题

### 端口被占用

将根目录 `.env` 中的 `LINGSHU_APP_PORT` 改为空闲端口，例如 `8081`，然后重新创建 API：

```powershell
docker compose --env-file .env -f Docker/docker-compose.yml up -d --force-recreate app
```

### API 一直处于 Created 状态

先检查两个一次性初始化任务：

```powershell
docker compose --env-file .env -f Docker/docker-compose.yml ps -a
docker compose --env-file .env -f Docker/docker-compose.yml logs knowledge-init workspace-init
```

两者都应以退出码 `0` 结束。修复 `.env`、Embedding 服务或 Docker Socket 权限后，再运行：

```powershell
docker compose --env-file .env -f Docker/docker-compose.yml up -d app
```

### 动态工作区创建失败

```powershell
docker compose --env-file .env -f Docker/docker-compose.yml exec app docker info
docker compose --env-file .env -f Docker/docker-compose.yml logs workspace-init
```

- Linux：确认 API 容器可访问 `/var/run/docker.sock`，且工作区目录权限正确。
- Windows：确认 `LINGSHU_WORKSPACE_DOCKER_HOST_ROOT` 是 Docker Desktop 可访问的绝对路径，且盘符已共享。

### 需要重建派生工作区镜像

```powershell
docker compose --env-file .env -f Docker/docker-compose.yml run --rm workspace-init
```

修改 `LINGSHU_WORKSPACE_BASE_IMAGE` 或 `LINGSHU_WORKSPACE_DOCKER_EXTRA_PIP` 后，派生镜像标签会变化并自动重建。

## 目录说明

```text
lingshu-Docker/
├── .env                         # 本机密钥与部署配置；不得提交
├── .env.example                 # 配置模板
├── Docker/
│   ├── docker-compose.yml        # 服务编排
│   ├── postgres-init/            # PostgreSQL 初始化 SQL
│   └── README.md                 # 编排细节
└── WorkDir/v2/                   # 动态工作区宿主机目录
```

## 数据与持久化

- 原始 `.RAG` 知识文件包含在 API 镜像中。
- 向量索引保存在 Qdrant Docker 命名卷中。
- 任务记忆、长期记忆、报告、PostgreSQL 与 MCP 数据也使用 Docker 命名卷。
- `docker compose down` 只停止容器，默认保留以上数据。

将本部署包复制到另一台机器时，原始知识文件会随 API 镜像获取；首次启动会自动重建该机器的 Qdrant 向量索引。

## 安全注意事项

- 仅用于已获得明确授权的安全测试。
- `.env` 含密钥与密码；已在 Git 中暂存时，请执行 `git restore --staged .env`，并确保 `.gitignore` 包含 `.env`。
- API 容器以 root 身份挂载 Docker Socket，以创建每会话工作区容器。这相当于容器拥有 Docker 管理权限，仅应在受信任环境部署；生产环境可考虑使用仅允许容器管理操作的 Socket Proxy。
- WebUI 默认只绑定到 `127.0.0.1`，不要在未配置访问控制的情况下直接公开端口；PostgreSQL、Qdrant、IDA MCP 与 AntSword 默认只在 Compose 内网暴露。

Compose 服务说明请见 [Docker/README.md](Docker/README.md)。
