# 灵枢（Lingshu）Docker 部署

灵枢是用于**授权安全测试**的多智能体系统。本仓库是独立的 Docker 部署包：应用代码、WebUI 与内置 `.RAG` 原始知识库均来自已发布的镜像，不需要 Python、Conda 或本地源码构建。

## 项目介绍

灵枢基于 AgentScope 2.x 构建，面向需要明确授权边界的安全测试与 CTF 分析任务。它的目标不是无约束的“自动攻击”，而是将大模型的开放式推理纳入一条**可审计、可恢复、以证据为依据且受人工闸门约束**的执行轨道。

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

只有 Orchestrator 可以推进任务状态；其他专家只返回证据和建议。这种“单调度者 + 专家分工”的拓扑避免多个 Agent 并发改写状态，也让决策顺序和责任主体可追溯。

### 核心能力

| 能力 | 说明 |
|---|---|
| 任务闭环 | 从目标理解、侦察、研判、人工审批、验证到报告生成，采用状态机约束合法阶段迁移。 |
| 授权与最小权限 | 网络命令需显式匹配授权目标；高风险攻击动作经过人工审批；不同专家仅获得职责所需工具。 |
| 证据与审计 | 任务状态、计划、工具调用、审批与终态均记录为 SHA-256 前向哈希审计链，可校验篡改。 |
| 三层记忆 | `.RAG` 安全知识库回答“是什么”，长期经验记忆回答“怎么做”，会话状态保存当前任务事实与上下文。 |
| 隔离执行 | 每个会话可创建独立 Kali Docker 工作区，支持安全工具、逆向分析与 MCP 扩展。 |
| 多模型与报告 | 支持多模型 Provider；任务结束后可生成结构化 Markdown、PDF 和 JSONL 结果。 |

### 安全治理原则

- 明确授权范围优先于模型自主判断；范围外目标与危险命令会被确定性规则拦截。
- 攻击阶段以人工审批和证据要求为闸门，不将模型文本视为成功事实。
- 角色隔离、严格参数校验、动作去重/熔断、上下文治理与审计链共同降低幻觉和越权风险。
- 系统内置约束是纵深防护，不能替代部署者的授权审查与法律责任。

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
