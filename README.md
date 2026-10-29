# FreeDeepseekAPI

<p align="center">
  <strong>面向 DeepSeek Web Chat 的本地 OpenAI 兼容 API 代理</strong>
</p>

<p align="center">
  <a href="https://github.com/csdjywyq/FreeDeepseekAPI/blob/main/LICENSE"><img alt="License MIT" src="https://img.shields.io/badge/license-MIT-green.svg" /></a>
  <img alt="Node.js 18 plus" src="https://img.shields.io/badge/node-18%2B-339933.svg" />
  <img alt="No npm dependencies" src="https://img.shields.io/badge/dependencies-0-blue.svg" />
  <img alt="OpenAI compatible" src="https://img.shields.io/badge/OpenAI-compatible-111111.svg" />
</p>

<p align="center">
  <a href="#-快速开始">快速开始</a> •
  <a href="#-功能特性">功能特性</a> •
  <a href="#-请求示例">请求示例</a> •
  <a href="#-模型">模型</a> •
  <a href="#-接口endpoints">接口</a> •
  <a href="#-open-webui">Open WebUI</a>
</p>

FreeDeepseekAPI 为 **DeepSeek Web Chat**（`chat.deepseek.com`）启动一个本地 API 服务，使 DeepSeek 网页版可以接入 Open WebUI、LiteLLM、Hermes、Claude Code、OpenAI SDK 风格客户端，以及其他 OpenAI 兼容工具。

项目使用你日常登录的 DeepSeek 账号，运行在独立的 Chrome 配置目录中。本地服务接收 API 请求，随后通过已保存的浏览器会话自行访问 DeepSeek Web。

> ⚠️ 这是一个实验性的网页聊天代理。DeepSeek 可能随时变更内部 Web API 而不作通知。生产环境建议使用 DeepSeek 官方付费 API。

ForgetMeAI: https://t.me/forgetmeai

---

## 导航

- [它的作用](#-它的作用)
- [功能特性](#-功能特性)
- [快速开始](#-快速开始)
- [Windows 启动](#-windows-启动)
- [Linux / Chromium 启动](#-linux--chromium-启动)
- [VPS / 无界面启动](#-vps--无界面启动)
- [VPS 常驻服务（systemd）](#-vps-常驻服务systemd)
- [Rootless Podman](#-rootless-podman)
- [诊断 / doctor（自检）](#-诊断--doctor自检)
- [会话复用与聊天重置](#-会话复用与聊天重置)
- [多账号池](#-多账号池)
- [命令行授权的设计思路](#-命令行授权的设计思路)
- [验证运行状态](#-验证运行状态)
- [请求示例](#-请求示例)
  - [Chat Completions](#chat-completions)
  - [推理（Reasoning）](#reasoning)
  - [联网搜索（Web search）](#web-search)
  - [流式输出（Streaming）](#streaming)
  - [Anthropic Messages API](#anthropic-messages-api)
  - [OpenAI Responses API](#openai-responses-api)
  - [工具调用（Tool calling）](#tool-calling)
- [模型](#-模型)
- [接口（Endpoints）](#-接口endpoints)
- [Open WebUI](#-open-webui)
- [更新登录状态](#-更新登录状态)
- [项目状态](#-项目状态)

---

## ✨ 它的作用

- 把 DeepSeek 网页版当作本地 API 端点使用。
- 将 DeepSeek 接入 Open WebUI 及其他 OpenAI 兼容客户端。
- 获取常规 JSON 响应或 SSE 流式响应。
- 使用推理模型，并单独返回 `reasoning_content`。
- 通过 Anthropic Messages API 适配层支持 Claude Code / Anthropic SDK。
- 通过 OpenAI Responses API 适配层支持新一代 OpenAI/Codex 风格客户端。
- 为不同 agent/用户保持彼此独立的网页会话。

## 🚀 功能特性

- **OpenAI 兼容 API：** `POST /v1/chat/completions`
- **Anthropic 兼容适配层：** `POST /v1/messages`
- **OpenAI Responses 适配层：** `POST /v1/responses`
- **流式输出：** SSE 分块，以及常规非流式 JSON 响应
- **推理输出：** 为思考型模型单独提供 `reasoning_content`
- **工具调用：** 解析 OpenAI tools、Anthropic tools 和 Responses function tools
- **模型能力：** `GET /v1/model-capabilities` 返回 alias → 实际网页模式
- **Agent 会话：** 每个 `user` / agent id 对应独立的 DeepSeek 会话
- **会话恢复：** 自动重置失效的链（chains）/会话（sessions）
- **零依赖：** 代理本体只需 Node.js 18+（`npm run auth` 需要 Node.js 22+），无 npm 依赖

---

## ⚡ 快速开始

```bash
git clone https://github.com/csdjywyq/FreeDeepseekAPI.git
cd FreeDeepseekAPI
npm run auth
npm start
```

`npm run auth` 会打开授权菜单：

1. 选择 `1`；
2. 在独立的 Chrome 配置目录中登录 DeepSeek；
3. 发送一条短消息，例如 `ok`；
4. 返回终端并按 Enter。

> `npm run auth` 使用 Node.js 内置的 WebSocket，因此需要 Node.js 22+。
> 代理本体在 Node.js 18+ 上即可运行；在较旧版本的 Node 上，请通过
> `npm run auth:import` 导入现成的 `deepseek-auth.json`。

`npm start` 会显示启动菜单：

- `1` —— 登录 / 更新 DeepSeek 登录状态
- `2` —— 导入授权文件 / cookies
- `3` —— 显示模型与状态
- `4` —— 启动代理（默认项，按 Enter）
- `5` —— 退出

配置可写入 `.env`（模板见 `.env.example`）：`npm start`、`npm run auth`
和 `npm run doctor` 都会自动读取该文件。环境变量的优先级高于 `.env`。

如需无界面/CI 方式启动，跳过菜单：

```bash
NON_INTERACTIVE=1 npm start
# 或
SKIP_ACCOUNT_MENU=1 npm start
```

服务默认监听：

```text
http://localhost:9655
```

默认情况下代理仅允许本机访问。若需从网络访问，请显式指定监听地址和独立的代理密钥：

```bash
HOST=0.0.0.0 PROXY_API_KEY='replace-with-a-long-random-value' npm start
```

之后以 `Authorization: Bearer <key>` 或 `x-api-key: <key>` 传递密钥
（Anthropic SDK 即采用后者）。未设置 `PROXY_API_KEY` 时，除健康检查外的
所有端点均无鉴权，因此不要把这样的实例暴露到网络上。

浏览器请求仅允许来自 loopback 源的调用。如果 UI 运行在其他地址上，
请用逗号追加其精确源（origin），例如
`PROXY_CORS_ORIGINS=https://ui.example.com,http://192.168.1.20:3000`。

---

## 🪟 Windows 启动

```powershell
git clone https://github.com/csdjywyq/FreeDeepseekAPI.git
cd FreeDeepseekAPI
npm run auth
npm start
```

如果 Chrome 安装在非默认位置，请显式指定路径：

```powershell
$env:CHROME_PATH="C:\Program Files\Google\Chrome\Application\chrome.exe"
npm run auth
```

若未找到 Chrome，`npm run auth` 现在会输出适用于 Windows/macOS/Linux 的
明确操作指引，而不是一段难以理解的调用栈。

---

## 🐧 Linux / Chromium 启动

```bash
git clone https://github.com/csdjywyq/FreeDeepseekAPI.git
cd FreeDeepseekAPI
CHROME_PATH=$(which chromium) npm run auth
npm start
```

如果 Chromium 的可执行文件名不同：

```bash
CHROME_PATH=$(which chromium-browser) npm run auth
# 或
CHROME_PATH=$(which google-chrome) npm run auth
```

---

## 🖥 VPS / 无界面启动

服务器上没有 Chrome 时最可靠的流程：

1. 在带图形界面/Chrome 的家用电脑上执行：

```bash
npm run auth
```

2. 将 `deepseek-auth.json` 复制到 VPS：

```bash
scp deepseek-auth.json user@your-vps:/opt/FreeDeepseekAPI/deepseek-auth.json
```

3. 在 VPS 上导入/校验文件，并设置安全的权限：

```bash
cd /opt/FreeDeepseekAPI
npm run auth:import -- --input ./deepseek-auth.json
npm run doctor -- --offline
```

4. 以非交互方式启动代理：

```bash
NON_INTERACTIVE=1 npm start
```

除现成的 `deepseek-auth.json` 外，也可以导入浏览器导出的 cookie：

```bash
DEEPSEEK_TOKEN="<token>" npm run auth:import -- --input ./cookies.json
```

> 注意：`deepseek-auth.json` 等同于你的 DeepSeek 网页登录凭证。不要提交到仓库、不要公开分享，并保持 `0600` 权限。

---

## 🛠 VPS 常驻服务（systemd）

不使用容器的方式：代理以独立用户身份作为系统服务运行，故障时自动重启，并随服务器一起启动。需要 Node.js 18+（`node` 位于 `/usr/bin` 或 `/usr/local/bin`）。

```bash
# 1. 用户与代码
sudo useradd --system --home /opt/FreeDeepseekAPI --shell /usr/sbin/nologin freedeepseek
sudo git clone https://github.com/csdjywyq/FreeDeepseekAPI /opt/FreeDeepseekAPI

# 2. 密钥：在家用电脑上执行 `npm run auth` 得到的授权文件，以及客户端密钥
sudo install -d -m 0750 -o root -g freedeepseek /etc/free-deepseek-api
sudo install -m 0600 -o freedeepseek -g freedeepseek ./deepseek-auth.json /etc/free-deepseek-api/deepseek-auth.json
openssl rand -hex 32 | sudo install -m 0600 -o freedeepseek -g freedeepseek /dev/stdin /etc/free-deepseek-api/proxy-api-key

# 3. 服务
sudo cp /opt/FreeDeepseekAPI/deploy/free-deepseek-api.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now free-deepseek-api

# 4. 验证
curl -fsS http://127.0.0.1:9655/readyz
sudo -u freedeepseek env DEEPSEEK_AUTH_PATH=/etc/free-deepseek-api/deepseek-auth.json node /opt/FreeDeepseekAPI/scripts/doctor.js
journalctl -u free-deepseek-api -f
```

- 服务仅监听 `127.0.0.1:9655`，并要求提供密钥
  （`Authorization: Bearer <密钥>` 或 `x-api-key`）。密钥保存在
  `/etc/free-deepseek-api/proxy-api-key`。
- 外部访问请走 HTTPS 反向代理（nginx/Caddy）或 SSH 隧道。
  如果需要直接监听网络，请在 `/etc/free-deepseek-api/env` 中设置
  `HOST=0.0.0.0`（任何 `DEEPSEEK_*` 配置也写在这里）。
- 更新：`cd /opt/FreeDeepseekAPI && sudo git pull && sudo systemctl restart free-deepseek-api`。
- 更换 DeepSeek 登录：用同一条命令放入新文件，以保留属主与权限，
  然后重启服务：
  `sudo install -m 0600 -o freedeepseek -g freedeepseek ./deepseek-auth.json /etc/free-deepseek-api/deepseek-auth.json && sudo systemctl restart free-deepseek-api`。

---

## 🦭 Rootless Podman

容器仅用于非交互式启动代理。请先在宿主机上执行 `npm run auth` 完成浏览器授权：
授权脚本和 `deepseek-auth.json` 都不会被打包进镜像。

请以普通用户身份运行 Podman，不要使用 `sudo`。

1. 构建本地镜像：

```bash
podman build --format docker --tag localhost/free-deepseek-api:local --file Containerfile .
```

2. 通过 Podman secrets 传入 DeepSeek 授权信息和独立的代理密钥：

```bash
podman secret create --replace free-deepseek-auth ./deepseek-auth.json

printf 'Proxy API key: '
IFS= read -r -s PROXY_API_KEY
printf '\n'
printf '%s' "$PROXY_API_KEY" |
  podman secret create --replace free-deepseek-proxy-key -
```

请使用足够长的随机密钥。该值只保留在当前 shell 的 `PROXY_API_KEY` 变量中，
用于测试 API；它不会进入镜像，也不会出现在 Podman 命令行里。

3. 以最小权限启动容器：

```bash
podman run --detach \
  --name free-deepseek-api \
  --publish 127.0.0.1:9655:9655 \
  --secret free-deepseek-auth,target=deepseek-auth.json,uid=1000,gid=1000,mode=0400 \
  --secret free-deepseek-proxy-key,target=proxy-api-key,uid=1000,gid=1000,mode=0400 \
  --read-only \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  localhost/free-deepseek-api:local
```

容器内已预设 `NON_INTERACTIVE=1`、`HOST=0.0.0.0` 以及两个 secret 的路径。
若密钥 secret 缺失或为空，`REQUIRE_PROXY_API_KEY=1` 会阻止容器启动。
在宿主机上端口只发布到 `127.0.0.1`；除非有独立的网络防火墙/访问策略，
否则不要去掉这个地址。

4. 检查存活状态、账号就绪状态和受保护的端点：

```bash
podman healthcheck run free-deepseek-api
curl --fail http://127.0.0.1:9655/readyz
curl --fail \
  -H "Authorization: Bearer $PROXY_API_KEY" \
  http://127.0.0.1:9655/v1/models
```

内置健康检查探测本地 `/health`（进程是否存活）。
构建时必须加 `--format docker`：在 OCI 格式下（Podman 的默认格式）
`HEALTHCHECK` 指令会被静默丢弃。
如果当前没有任何 DeepSeek 授权账号可处理请求，`/readyz` 会额外返回 `503`。
容器诊断：

```bash
podman logs free-deepseek-api
podman inspect --format '{{.State.Health.Status}}' free-deepseek-api
```

停止并删除容器，同时清除已保存的 Podman secrets：

```bash
podman stop free-deepseek-api
podman rm free-deepseek-api
podman secret rm free-deepseek-auth free-deepseek-proxy-key
unset PROXY_API_KEY
```

轮换授权信息或代理密钥时，请替换对应的 secret 并重建容器，
以免行为受 Podman 版本差异影响。

---

## 🩺 诊断 / doctor（自检）

```bash
npm run doctor
# 不向 DeepSeek 发起网络请求：
npm run doctor -- --offline
```

`doctor` 会检查：

- 是否找到 `deepseek-auth.json` / `DEEPSEEK_AUTH_DIR`；
- JSON 是否有效；
- 是否包含 `token`、`cookie`、`wasmUrl`；
- 在 macOS/Linux 上文件权限是否安全（`0600`）；
- 常规启动时，DeepSeek PoW 端点是否可用。

如果遇到 `data.biz_data is null`、`fetch failed`、`401/403/429`，或 Hermes/OpenCode
看不到模型，请先执行 `npm run doctor`。

---

## ♻️ 会话复用与聊天重置

FreeDeepseekAPI 不会无理由地为每个 HTTP 请求新建 DeepSeek 聊天。逻辑如下：

- 一个 `x-agent-session`、`session` 或 `user` 对应一个 DeepSeek 聊天会话；
- 如果会话 id 已存在，代理会复用该会话，并通过 `parent_message_id` 延续消息链；
- Agent 类客户端（Hermes、Cline、Claude Code、OpenCode 等）会在每次请求中带上完整历史。
  代理会记住哪些消息已经存在于 DeepSeek 聊天中，只向在线聊天发送**新增内容**
  （工具结果、新问题）以及一段关于工具调用格式的简短提示。此前每一步都会重发
  整个对话（最多 80 000 字符），聊天上下文呈平方级增长，十来步之后 DeepSeek
  开始返回空内容、提示“内容过长”，或不再调用工具（#23、#30）；
- 若客户端改动了历史（压缩/编辑）、更换了 system prompt 或工具集合，
  则会以完整提示词新建一个 DeepSeek 聊天；
- 触发自动重置的情形：TTL 到期、DeepSeek 会话报错、消息链过长，
  或聊天内累计字符数超过 `DEEPSEEK_MAX_SESSION_CHARS`
  （默认为 3 × `DEEPSEEK_MAX_PROMPT_CHARS`）；
- 同一 agent 的并发请求（例如 Hermes 中的后台 review）不会混在同一个聊天里：
  第二个请求会在另一个临时 DeepSeek 聊天中处理；
- 本地历史以精简上下文形式保存，便于新的 DeepSeek 会话接着对话。
- 发送前，较长的 agent 请求会按 `DEEPSEEK_MAX_PROMPT_CHARS`（默认 80 000 字符）
  截断：保留任务开头、最新的工具结果和工具适配说明；
- 若客户端已提供多轮历史，则不会重复附加本地恢复历史；
- 空响应最多重试 `DEEPSEEK_MAX_RETRIES` 次（默认 2），且每次重试都会缩减上下文。
- 长时间生成（思考）不会因超时被中断：流式输出受“无数据停顿”上限控制
  （`DEEPSEEK_STREAM_IDLE_TIMEOUT_MS`，默认 60 秒）和总时长上限
  `DEEPSEEK_STREAM_MAX_MS`（10 分钟）。

显式指定 agent/会话：

```bash
curl -X POST http://localhost:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-agent-session: my-agent" \
  -d '{"model":"deepseek-chat","messages":[{"role":"user","content":"你好"}]}'
```

查看活跃会话：

```bash
curl http://localhost:9655/v1/sessions
```

重置单个会话：

```bash
curl -X POST "http://localhost:9655/reset-session?agent=my-agent"
```

重置所有会话：

```bash
curl -X POST "http://localhost:9655/reset-session?agent=all"
```

为什么 DeepSeek 网页里仍然会出现聊天：代理走的是内部 Web Chat API，
而真实的聊天会话由 DeepSeek 服务端保存。这对网页代理来说是正常现象。
会话复用的目标只是避免无必要地新建聊天，并在消息链过期/损坏时干净地重置。

---

## 👥 多账号池

可以接入多个授权文件。正确的模型是「按 agent/会话粘性绑定账号」——
代理不会在活跃的 DeepSeek 会话中途切换账号。若某账号返回 `401/403/429`
并进入冷却，会话会被安全重置，新请求可转到另一个可用账号。

方式一 —— 用目录存放授权文件：

```bash
mkdir -p accounts
cp deepseek-auth-main.json accounts/main.json
cp deepseek-auth-backup.json accounts/backup.json
chmod 600 accounts/*.json
DEEPSEEK_AUTH_DIR=./accounts NON_INTERACTIVE=1 npm start
```

方式二 —— 直接给文件列表：

```bash
DEEPSEEK_AUTH_PATH="./accounts/main.json,./accounts/backup.json" NON_INTERACTIVE=1 npm start
```

账号池的工作方式：

- 新 agent/会话按轮询（round-robin）分配可用账号；
- 选中的账号会粘性绑定到该会话（`sticky`）；
- 遇到 `401`、`403`、`429` 时账号进入冷却，请求改用另一个就绪账号重试；
- DeepSeek 返回「Too many messages」同样视为限流：账号进入短暂冷却
  （`DEEPSEEK_RATE_LIMIT_COOLDOWN_MS`，默认 60 秒）；若已无其他账号，
  客户端会收到带 `Retry-After` 的 `429`；
- 若某会话的粘性账号进入冷却，旧的 DeepSeek 会话会被重置，
  以免继续冲击已限流/失效的账号；
- `/health` 中可查看账号状态，但不暴露授权文件路径和文件名；
- 授权文件必须以 `0600` 权限保存。

调整冷却时间：

```bash
DEEPSEEK_ACCOUNT_COOLDOWN_MS=600000 npm start
```

---

## 🔑 命令行授权的设计思路

PR #3 中的密码登录流程可以实现，但更安全的做法是不保存密码、也不把它设为默认。规范实现应当：

1. `npm run auth:console` 通过隐藏输入提示获取邮箱/手机号与密码。
2. 密码只保留在进程内存中，不写入文件/日志/history。
3. 脚本通过 `fetch`/CDP 复现网页登录流程：获取验证码/校验挑战，
   把链接或验证码交给人工处理，等待确认。
4. 登录成功后，仅保存标准格式的 `deepseek-auth.json`。
5. 若 DeepSeek 要求验证码/双因素认证，脚本应如实提示“请打开链接完成验证后按 Enter”，
   而不是尝试绕过防护。
6. VPS 场景建议使用 `auth:console --no-save-password --output deepseek-auth.json`。

最小安全 MVP：命令行授权仅做交互式，不支持环境变量传密码。可接受的自动化方案：
`DEEPSEEK_EMAIL=... npm run auth:console`，但密码仍通过隐藏输入提示获取。

---

## ✅ 验证运行状态

```bash
curl http://localhost:9655/
curl http://localhost:9655/v1/models
curl http://localhost:9655/v1/model-capabilities
```

一切正常时，`/health` 会返回服务状态、支持的 alias 列表和 `config_ready: true`。

---

## 🧪 请求示例

### Chat Completions

```bash
curl -X POST http://localhost:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "messages": [{"role": "user", "content": "你好！用一句话回答。"}],
    "stream": false
  }'
```

### Reasoning

```bash
curl -X POST http://localhost:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-reasoner",
    "messages": [{"role": "user", "content": "简短回答：天空为什么是蓝色的？"}],
    "stream": false
  }'
```

当 `stream: true` 时，普通响应（请求中不含 `tools`）会随 DeepSeek 的生成进度
实时返回给客户端：先思考内容，后正文。若请求中包含 `tools`，响应可能是一个
工具调用，因此代理会等到结束再整体返回；等待期间每隔
`DEEPSEEK_STREAM_KEEPALIVE_MS`（10 秒）发送一条 SSE 注释 `: keep-alive`，
避免客户端因超时断开连接。流开始之后发生的错误会以流内的 `error` 事件下发
（OpenAI 与 Anthropic 的 SDK 会将其转为异常）；流开始之前的错误则以常规
HTTP 状态码返回。

对推理模型，API 会将思考链与最终回答分开返回：

- 非流式：`choices[0].message.reasoning_content`
- 流式：`choices[0].delta.reasoning_content`
- 用量：`usage.completion_tokens_details.reasoning_tokens`

`reasoning_tokens` 是基于从 DeepSeek Web 提取的 `THINK` 文本做出的近似估算，
因为网页流并不会单独给出官方的推理 token 用量。

### Web search

```bash
curl -X POST http://localhost:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat-search",
    "messages": [{"role": "user", "content": "查一条关于 DeepSeek 的最新事实，简短回答。"}],
    "stream": false
  }'
```

### Streaming

```bash
curl -N -X POST http://localhost:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "messages": [{"role": "user", "content": "讲一个简短的笑话。"}],
    "stream": true
  }'
```

### Anthropic Messages API

```bash
curl -X POST http://localhost:9655/v1/messages \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "max_tokens": 512,
    "messages": [{"role": "user", "content": "只回复 OK"}],
    "stream": false
  }'
```

Claude Code 可直接指定后端：

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:9655"
export ANTHROPIC_AUTH_TOKEN="dummy-key"
export CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1
claude --model deepseek-chat
```

### OpenAI Responses API

```bash
curl -X POST http://localhost:9655/v1/responses \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "input": "只回复 OK",
    "stream": false
  }'
```

### Tool calling

FreeDeepseekAPI 接受：

- OpenAI `tools`；
- Anthropic `tools`；
- Responses API function tools。

代理要求 DeepSeek 返回严格的 JSON 工具调用，同时也支持解析以下兜底格式：

- `TOOL_CALL:`
- 带显式 `tool_call`、`tool_calls` 或 `function_call` 外层结构的围栏 JSON
- `<tool_call>...</tool_call>`
- DeepSeek DSML（`<｜DSML｜tool_calls>...`）以及网页版变体 `<｜｜DSML｜｜ Tool Calls>`

解析时会修正模型的典型错误：Windows 路径中未转义的反斜杠
（`"C:\git\src\index.ts"`）以及字符串参数中的真实换行。
标注了其他语言的代码块（```` ```csharp ````、```` ```python ````）永远不会被当作工具调用。

---

## 🧠 模型

`GET /v1/models` 只返回当前已通过本代理验证可用的 alias。

### DeepSeek Web 统一模式（自 2026-09-10 起）

2026 年 9 月 10 日，DeepSeek 在网页聊天中把「快速」「专家」和「识别」三种模式
合并为基于 **DeepSeek-V4.1-Flash** 的单一智能模式，并于 9 月 14 日下线 V4 Pro。
模型切换器已不存在，仅保留「深度思考」（`thinking_enabled`）和「搜索」
（`search_enabled`）两项开关（#31）。

因此所有 alias 现在都向 Web API 发送相同的 `model_type: "default"`，
区别只在于这两个开关。旧名称（`deepseek-expert`、`deepseek-v4-pro` 等）被保留下来
以免破坏客户端配置，并在 `/v1/model-capabilities` 中标记为 `deprecated`。
如果 DeepSeek 再次更改 `model_type` 的取值，可通过 `DEEPSEEK_MODEL_TYPE` 覆盖
（空值表示不发送该字段）。

### 可用 alias

| Alias | 深度思考 | 联网搜索 | 备注 |
| --- | --- | --- | --- |
| `deepseek-chat` | 否 | 否 | 基础聊天 |
| `deepseek-default` | 否 | 否 | 兼容 alias |
| `deepseek-v4-flash` | 否 | 否 | 官方 API 名称 |
| `deepseek-v4.1-flash` | 否 | 否 | 当前网页聊天模型 |
| `deepseek-v3` | 否 | 否 | 已废弃 alias |
| `deepseek-reasoner` | 是 | 否 | `thinking_enabled=true` |
| `deepseek-r1` | 是 | 否 | R1 兼容 alias |
| `deepseek-chat-search` | 否 | 是 | 联网搜索 |
| `deepseek-default-search` | 否 | 是 | 联网搜索 alias |
| `deepseek-reasoner-search` | 是 | 是 | 推理 + 搜索 |
| `deepseek-r1-search` | 是 | 是 | R1 兼容 + 搜索 |
| `deepseek-expert` | 是 | 否 | 已废弃：「专家」已并入统一模式 |
| `deepseek-v4-pro` | 是 | 否 | 已废弃：V4 Pro 已下线，实际由 V4.1-Flash 应答 |
| `deepseek-expert-search` | 是 | 是 | 已废弃 alias，当前仍可用 |

### 图片（已在真实 DeepSeek Web API 上验证）

2026 年 10 月 7 日，已在真实 DeepSeek Web API 上验证通过 `deepseek-vision`
传输 PNG/JPEG：multipart 上传、文件处理从 `PENDING` 经 `PARSING` 到 `SUCCESS`、
在 completion 中引用 file ID，以及返回包含可读文字、颜色与布局的视觉回答。
对于近似正方形的矩形，模型会称之为正方形，因此形状识别的准确度需按具体任务验证。
图片上传默认开启。若要禁止向 DeepSeek 发送图片，请显式设置
`DEEPSEEK_IMAGE_UPLOAD=0`：

```bash
DEEPSEEK_IMAGE_UPLOAD=0 npm start
```

- 支持内联图片：OpenAI `image_url` / Responses 的
  `input_image`（`data:image/...;base64,...`），以及 Anthropic 的 `image`
  （`source.type: "base64"`，包括位于 `tool_result` 内的情况）；
- `https://...` 链接不会被下载（防 SSRF）——模型只能看到链接地址；
- 支持 PNG、JPEG、WebP 和 GIF；其他格式（例如 SVG）会退化为文本标记；
- 已打开的聊天只上传新消息中的图片；新聊天则上传对话中最近
  `DEEPSEEK_MAX_IMAGES`（4）张图片；单张图片上限
  `DEEPSEEK_MAX_IMAGE_BYTES`（7 MB），整个请求上限 10 MB；已上传过的图片
  不会重复上传（缓存 30 分钟）；
- 若 DeepSeek 无法处理该图片，请求会以 `502 image_upload_failed` 失败，
  而不会悄悄返回“盲答”；
- 开启上传后，`deepseek-vision` 会出现在 `/v1/models` 中
  （它同样是那个统一模式）。

当 `DEEPSEEK_IMAGE_UPLOAD=0` 时，图片会被替换为简短标记
`[Image attached; not visible through this proxy]`，其字节内容不会上传到 DeepSeek。

完整映射：

```bash
curl http://localhost:9655/v1/model-capabilities
```

---

## 🔌 接口（Endpoints）

| Method | Path | 用途 |
| --- | --- | --- |
| `GET` | `/` 或 `/health` | 代理状态 |
| `GET` | `/v1/models` | 可用的 OpenAI 兼容 alias 列表 |
| `GET` | `/v1/model-capabilities` | 完整映射：alias、真实模型、能力 |
| `POST` | `/v1/chat/completions` | OpenAI 兼容 Chat Completions |
| `POST` | `/v1/messages` | Anthropic Messages API 适配层 |
| `POST` | `/v1/responses` | OpenAI Responses API 适配层 |
| `GET` | `/v1/sessions` | 活跃的本地 agent 会话 |
| `POST` | `/reset-session?agent=<id>` | 重置单个会话 |
| `POST` | `/reset-session?agent=all` | 重置所有会话 |

---

## 🖥 Open WebUI

Docker 中运行 Open WebUI 时的 Base URL：

```text
http://host.docker.internal:9655/v1
```

本地无 Docker 运行时：

```text
http://localhost:9655/v1
```

若未设置 `PROXY_API_KEY`，API key 可填任意值。若已设置密钥，客户端必须传入
该密钥——代理在允许访问模型、会话和 completions 之前会校验 bearer token。

---

## 🔐 更新登录状态

```bash
npm run auth
npm start
```

如果 DeepSeek 开始返回 `401`、`403`，或要求新的 PoW/会话，请重新执行
`npm run auth` 以更新已保存的浏览器会话。

本地授权文件不应进入 GitHub：

- `deepseek-auth.json`
- `.chrome-for-testing-profile-deepseek/`
- `.env`

它们已被加入 `.gitignore`。

---

## 🧪 测试

语法检查、单元测试，以及面向本地 DeepSeek mock 服务端的端到端测试
（无需网络和账号；`node --test` 需要 Node.js 18.13+）：

```bash
npm test
```

面向已启动本地代理的实时冒烟测试（若设置了 `PROXY_API_KEY`，会从环境变量读取）：

```bash
BASE_URL=http://127.0.0.1:9655 MODEL=deepseek-chat npm run test:live
```

---

## 📌 项目状态

FreeDeepseekAPI 是一个用于本地使用与集成的实验性网页聊天代理。它依赖 DeepSeek Web Chat 当前的接口约定，因此当 DeepSeek 侧发生变更时，可能需要更新授权/会话逻辑或模型映射。

如果出现异常：

1. 通过 `npm run auth` 更新登录状态；
2. 检查 `/v1/model-capabilities`；
3. 在新会话上重试请求；
4. 若问题仍然存在——很可能是 DeepSeek 更改了内部 Web API。

---

<p align="center">
  <strong>ForgetMeAI</strong> · <a href="https://t.me/forgetmeai">Telegram</a>
</p>
