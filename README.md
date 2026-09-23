# dify-wsl2-deploy

# dify-wsl2-deploy全过程

## 这是什么

这是一个在 **Windows 10 LTSC + WSL2 (Ubuntu)** 环境下，使用 **Docker/Podman** 部署 **Dify 知识库平台**的完整操作记录。内容涵盖从源码获取、环境配置、容器启动，到 RAG 工作流搭建、端口迁移（8001 → 8080）以及常见踩坑与解决方案。

## 解决了什么问题

- **Windows 下部署 Dify 的门槛问题**：通过 WSL2 运行 Linux 容器，避免 Windows 原生环境的兼容性困扰。
- **WSL2 端口访问不稳定**：通过固定 `localhost` 访问、配置 Windows 端口转发，解决 IP 变动与连接拒绝问题。
- **本地算力不足**：将 Embedding 模型从本地 `bge-m3` 切换到云端 `BAAI/bge-m3`，检索耗时从 7 秒降至 0.2 秒。
- **RAG 工作流踩坑**：记录了 Agent 死循环、Tavily JSON 解析失败、变量未定义等问题的排查与修复过程。
- **部署后的长期维护**：给出容器自启动、定期备份、知识库文档规范等实用建议。

## 怎么用

下面按步骤操作即可。环境准备 → 部署 Dify → 配置 RAG 工作流 → 端口迁移 → 日常维护。每一步都附有命令与截图说明（文字版）。如遇问题，可查阅「踩坑记录」章节。

---

## 环境准备

- **操作系统**：Windows 10 IoT 企业版 LTSC 21H2
- **WSL2**：已启用，发行版 Ubuntu
- **容器运行时**：WSL2 内安装 Docker 或 Podman
- **硬件**：普通笔记本，16GB RAM
- **网络**：WSL2 与 Windows 主机互通

## 部署步骤

### 1. 获取 Dify 源码

```bash
cd ~
git clone https://github.com/langgenius/dify.git
cd dify/docker
```

### 2. 配置 `.env`

```bash
cp .env.example .env
nano .env
```

关键变量：

```ini
EXPOSE_NGINX_PORT=8001
CONSOLE_API_URL=
CONSOLE_WEB_URL=
SERVICE_API_URL=
APP_API_URL=
APP_WEB_URL=
```

> **注意**：若后续要改端口，必须同步修改 `EXPOSE_NGINX_PORT` 和上述 URL 变量。

### 3. 启动 Dify

```bash
docker compose up -d
```

检查容器状态：

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### 4. 访问 Dify

- 本机：`http://localhost:8001`
- 局域网：`http://<Windows主机IP>:8001`
- WSL2 虚拟 IP：`http://<WSL2_IP>:8001`（IP 会随 WSL2 重启变化，不推荐长期使用）

首次进入需设置管理员账号。

## RAG 知识库配置

### 知识库创建

- 上传 `.md` 文件
- 分段模式：自定义，标识符 `##`
- 索引方式：高质量（需 Embedding 模型）

### Embedding 模型选择

- **本地**：`bge-m3:latest`（在普通笔记本上计算慢，易超时）
- **云端**：硅基流动 `BAAI/bge-m3`（API Base: `https://api.siliconflow.cn/v1`，免费额度 1000万 tokens/月）
- **推荐**：使用云端 Embedding，检索速度从 7 秒降至 0.2 秒

### 工作流节点连线

```
开始 → 知识检索 → 条件分支
    ├─ IF (结果为空) → Tavily Search → 代码执行(Python清洗JSON) → LLM → 直接回复
    └─ ELSE (结果非空) → LLM → 直接回复
```

### 防幻觉提示词

```
你是一个严谨的AI助手。请仅根据以下上下文回答问题：
{{上下文}}
如果上下文没有相关信息，请直接回答“根据现有资料无法回答”，不要编造。
用户问题：{{query}}
```

## 踩坑记录

| 问题 | 现象 | 原因 | 解决方法 |
|------|------|------|----------|
| Agent 节点死循环 | 耗时 11.8s，消耗 11866 Tokens | 云端模型与 Function Calling 兼容性差 | 弃用 Agent，改用条件分支 + 固定节点 |
| 变量未定义 | `Variable key ... not found` | 手动输入 `{{context}}` 未识别 | 用 `{x}` 按钮选择变量 |
| 本地向量检索超时 | 知识库有文档却走 Tavily 分支 | 本地 bge-m3 计算慢 | 切换云端 `BAAI/bge-m3`，耗时降至 0.2s |
| Tavily JSON 导致拒答 | 搜索成功但 LLM 回复“无法回答” | Tavily 返回 Array[Object] | 加 Python 代码节点，提取 `content` 拼接 |
| 发布检查报错 | Rerank 模型为空 | 误开 Rerank 或冗余空变量 | 关闭 Rerank；删除冗余变量 |
| Web 分享链接报错 | `ERR_CONNECTION_REFUSED` | 分享链接缺少端口号 | 手动加 `:8001` |
| WSL2 重启后容器消失 | `docker ps` 无容器 | 未设置 `restart: always` | 添加 `--restart always` |
| 端口映射不生效 | `docker port` 无输出 | 容器启动后端口绑定延迟 | 等待 1-2 分钟，或 `docker compose down && up -d` |

## 修改端口（8001 → 8080）

### 备份配置

```bash
cd ~/dify/docker
cp .env .env.bak_20260920
cp docker-compose.yaml docker-compose.yaml.bak_20260920
```

### 检查端口占用

Windows：
```cmd
netstat -ano | findstr :8080
```

WSL2：
```bash
sudo ss -tulnp | grep 8080
```

### 修改 `.env`

```ini
EXPOSE_NGINX_PORT=8080
CONSOLE_API_URL=http://localhost:8080
CONSOLE_WEB_URL=http://localhost:8080
SERVICE_API_URL=http://localhost:8080
APP_WEB_URL=http://localhost:8080
NGINX_HTTPS_ENABLED=false
```

### 重启并验证

```bash
docker compose down
docker compose up -d
docker port docker-nginx-1
```

应显示 `0.0.0.0:8080->80/tcp`。

### 失败还原

```bash
cp .env.bak_20260920 .env
cp docker-compose.yaml.bak_20260920 docker-compose.yaml
docker compose down && docker compose up -d
```

## 长期使用建议

1. 固定用 `localhost` 访问，避免 WSL2 IP 变动。
2. 设置容器自启动：`--restart always`。
3. Embedding 走云端：本地算力有限，云端免费且快。
4. 知识库文档规范：`.md` 格式，分段符 `##`。
5. 定期备份：`.env`、`docker-compose.yaml`、知识库文档。
6. 安全提醒：不要将内网 IP 暴露到公网。

## 许可

本仓库的部署脚本和文档采用 **MIT License** 发布。

Dify 社区版采用 **修改版 Apache License 2.0**，附加条件如下：

1. **允许商业使用**，包括作为其他应用的后端服务或企业应用开发平台。
2. **多租户限制**：除非获得 Dify 书面授权，不得使用 Dify 源码运营多租户环境（一个租户对应一个工作空间）。
3. **LOGO 与版权信息**：使用 Dify 前端时，不得移除或修改 Dify 控制台或应用中的 LOGO 或版权信息。
4. 除上述附加条件外，其余权利与限制遵循 Apache License 2.0。

MIT License

Copyright (c) 2026 Hot-Tail

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```
