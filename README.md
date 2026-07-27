# kiro2cc-proxy

[![Tests](https://github.com/TsinHzl/kiro2cc-proxy/actions/workflows/test.yaml/badge.svg)](https://github.com/TsinHzl/kiro2cc-proxy/actions/workflows/test.yaml)
[![codecov](https://codecov.io/gh/TsinHzl/kiro2cc-proxy/graph/badge.svg)](https://codecov.io/gh/TsinHzl/kiro2cc-proxy)

一个用 Rust 编写的 Anthropic Claude API 兼容代理服务，将 Anthropic API 请求转换为 Kiro API 请求。

> **✅ 支持模型：Claude Sonnet 5 / Claude Sonnet 4.5 / Claude Sonnet 4.6 / Claude Opus 4.5 / Claude Opus 4.6 / Claude Opus 4.7 / Claude Opus 4.8 / Claude Haiku 4.5 / DeepSeek 3.2 / GLM-5 / MiniMax M2.1 / MiniMax M2.5 / Qwen3-Coder / GPT-5.6 Sol / GPT-5.6 Terra / GPT-5.6 Luna**

[English](README.en.md) | 中文

## 免责声明

本项目仅供研究使用，Use at your own risk，使用本项目所导致的任何后果由使用人承担，与本项目无关。本项目与 AWS/KIRO/Anthropic/Claude 等官方无关，不代表官方立场。

## 功能特性

- **Anthropic API 兼容**：完整支持 Anthropic Claude API 格式
- **流式响应**：支持 SSE (Server-Sent Events) 流式输出
- **Token 自动刷新**：自动管理和刷新 OAuth Token
- **多账号支持**：支持配置多个账号，按优先级自动故障转移
- **负载均衡**：支持 `priority`（按优先级）和 `balanced`（均衡分配）两种模式
- **智能重试**：单账号最多重试 3 次，单请求最多重试 9 次
- **Thinking 模式**：支持 Claude 的 extended thinking 功能
- **工具调用**：完整支持 function calling / tool use
- **WebSearch**：内置 WebSearch 工具转换逻辑
- **Admin 管理**：可选的 Web 管理界面，支持账号管理、余额查询等
- **账号级代理**：支持为每个账号单独配置 HTTP/SOCKS5 代理

---

## 目录

- [快速开始](#快速开始)
- [Docker 部署](#docker-部署)
- [获取 Kiro 账号](#获取-kiro-账号)
- [配置详解](#配置详解)
- [接入 Claude Code](#接入-claude-code)
- [API 端点](#api-端点)
- [模型映射](#模型映射)
- [Admin 管理面板](#admin-管理面板)
- [常见问题](#常见问题)
- [注意事项](#注意事项)

---

## 快速开始

**这个项目是什么？**

kiro2cc-proxy 是一个代理服务。它把标准的 Anthropic Claude API 请求转发给 Kiro（AWS 的 AI 编程工具），让你可以用 Claude Code 使用 Kiro 账号的模型。

>  一句话说明白，就是：它能把登录的 Kiro 账号上的模型代理到 Claude Code 上进行使用。否则的话就只能在 Kiro IDE 或者 Kiro CLI 上使用。

**使用前提：**

1. 拥有一个 Kiro 账号（通过 [kiro.dev](https://kiro.dev) 注册，支持 Social 登录）
2. 从 Kiro IDE 或账号管理工具中导出账号（`refreshToken` 等信息）
3. > ⚠️ **【重要】国内用户**：必须配置本地 HTTP/SOCKS5 代理（Clash/V2Ray 等），否则所有 Claude 模型请求均会返回 `INVALID_MODEL_ID` 错误，无法使用。

**整体流程：**

```
安装 Docker → 部署服务 → 填入账号 → 配置客户端
```

---

## Docker 部署

### 前置要求

- 安装 Docker 和 Docker Compose
- 一个或多个 Kiro 账号的 `refreshToken`

### 快速部署

**1. 克隆项目**

```bash
git clone https://github.com/kch-space/kiro-proxy.git
cd kiro-proxy
```

**2. 创建数据目录**

```bash
mkdir -p data
```

**3. 创建配置文件**

创建 `data/config.json`：

```bash
cat > data/config.json << 'EOL'
{
  "apiKey": "your-api-key-here",
  "host": "0.0.0.0",
  "port": 5678,
  "adminApiKey": "your-admin-key-here",
  "proxyUrl": "",
  "balanceMode": "priority",
  "tlsBackend": "rustls"
}
EOL
```

创建 `data/credentials.json`（初始为空数组）：

```bash
echo "[]" > data/credentials.json
```

**4. 启动服务**

```bash
docker compose up -d
```

**5. 查看日志**

```bash
docker compose logs -f
```

**6. 访问管理面板**

打开浏览器访问 `http://localhost:5678/admin`，使用 `config.json` 中配置的 `adminApiKey` 登录。

### 配置说明

**环境变量（可选）**

也可以通过环境变量配置，在 `docker-compose.yml` 中添加：

```yaml
services:
  kiro2cc-proxy:
    environment:
      - API_KEY=your-api-key-here
      - ADMIN_API_KEY=your-admin-key-here
      - HOST=0.0.0.0
      - PORT=5678
```

**国内用户代理配置**

> ⚠️ **重要**：国内用户必须配置代理才能访问 Claude 模型

在 `data/config.json` 中添加 `proxyUrl`：

```json
{
  "proxyUrl": "http://host.docker.internal:7890"
}
```

`host.docker.internal` 会自动解析为宿主机 IP（需要 Docker 18.03+）。

### 更新服务

```bash
cd kiro-proxy
git pull
docker compose pull
docker compose down && docker compose up -d
```

### 停止服务

```bash
docker compose down
```

### 完全清理

```bash
docker compose down -v
rm -rf data
```

---

## 获取 Kiro 账号

### 方式一：通过 Kiro IDE（推荐）

1. 安装并登录 [Kiro IDE](https://kiro.dev)
2. 打开命令面板（macOS：`Cmd+Shift+P` / Windows：`Ctrl+Shift+P`）
3. 输入 `Kiro: Export Credentials`
4. 选择账号后，JSON 格式的账号信息会被复制到剪贴板

### 方式二：通过浏览器开发者工具

1. 访问 [kiro.dev](https://kiro.dev) 并登录
2. 打开浏览器开发者工具（F12）
3. 切换到 Console 标签页
4. 执行以下脚本：

```javascript
(function() {
  const tokens = JSON.parse(localStorage.getItem('kiro_tokens') || '{}');
  const config = {
    name: "My Kiro Account",
    refreshToken: tokens.refreshToken,
    apiRegion: "us-east-1",
    priority: 1
  };
  console.log(JSON.stringify(config, null, 2));
  navigator.clipboard.writeText(JSON.stringify(config, null, 2));
  alert('账号信息已复制到剪贴板');
})();
```

### 账号配置格式

```json
{
  "name": "账号名称",
  "refreshToken": "refresh_token_value",
  "apiRegion": "us-east-1",
  "priority": 1,
  "proxyUrl": ""
}
```

**字段说明：**

- `name`: 账号名称（便于识别）
- `refreshToken`: Kiro 账号的 refresh token（必需）
- `apiRegion`: API 区域，默认 `us-east-1`（可选）
- `priority`: 优先级，数字越小优先级越高（可选，默认 1）
- `proxyUrl`: 该账号专用的代理地址（可选）

### 添加账号

**方式一：通过管理面板**

1. 访问 `http://localhost:5678/admin`
2. 使用 `adminApiKey` 登录
3. 点击「添加账号」按钮
4. 填入账号信息并保存

**方式二：手动编辑配置文件**

编辑 `data/credentials.json`：

```json
[
  {
    "name": "Main Account",
    "refreshToken": "your_refresh_token_1",
    "apiRegion": "us-east-1",
    "priority": 1
  },
  {
    "name": "Backup Account",
    "refreshToken": "your_refresh_token_2",
    "apiRegion": "us-west-2",
    "priority": 2
  }
]
```

编辑后重启服务：

```bash
docker compose restart
```

---

## 配置详解

### config.json 配置项

| 字段 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `apiKey` | string | 是 | - | 客户端访问服务的 API Key |
| `host` | string | 否 | `127.0.0.1` | 监听地址，Docker 部署建议 `0.0.0.0` |
| `port` | number | 否 | `5678` | 监听端口 |
| `adminApiKey` | string | 否 | - | 管理面板的 API Key |
| `proxyUrl` | string | 否 | - | 全局代理地址（如 `http://127.0.0.1:7890`） |
| `balanceMode` | string | 否 | `priority` | 负载均衡模式：`priority` 或 `balanced` |
| `tlsBackend` | string | 否 | `rustls` | TLS 后端：`rustls` 或 `native-tls` |

### credentials.json 配置项

每个账号对象包含以下字段：

| 字段 | 类型 | 必需 | 默认值 | 说明 |
|------|------|------|--------|------|
| `name` | string | 是 | - | 账号名称 |
| `refreshToken` | string | 是 | - | Kiro 账号的 refresh token |
| `apiRegion` | string | 否 | `us-east-1` | API 区域 |
| `priority` | number | 否 | `1` | 优先级（数字越小越优先） |
| `proxyUrl` | string | 否 | - | 该账号专用代理 |
| `profileArn` | string | 否 | - | 企业版 IdC 账号需要 |

### 负载均衡模式

**priority 模式（默认）**

按 `priority` 字段排序，优先使用优先级高的账号。当前账号失败时，自动切换到下一优先级账号。

**balanced 模式**

轮询所有可用账号，实现负载均衡。适合多账号均衡使用的场景。

---

## 接入 Claude Code

### 第一步：安装 Claude Code

从 [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=Anthropic.claude-code) 或 IDE 插件市场安装 Claude Code 插件。

### 第二步：配置 API

在 Claude Code 设置中：

1. **API Provider**: 选择 `Anthropic`
2. **API Key**: 填入你在 `config.json` 中设置的 `apiKey`
3. **Base URL**: 填入代理服务地址
   - 本地部署：`http://localhost:5678`
   - 远程部署：`http://your-server-ip:5678`

### 第三步：开始使用

配置完成后，Claude Code 会通过代理服务访问 Kiro 账号的模型。

---

## API 端点

### 核心端点

| 端点 | 说明 |
|------|------|
| `POST /v1/messages` | Anthropic 兼容的消息 API |
| `POST /v1/chat/completions` | OpenAI 兼容的聊天 API（实验性） |

### 管理端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/admin/credentials` | GET | 获取所有账号 |
| `/api/admin/credentials` | POST | 添加账号 |
| `/api/admin/credentials/:id` | DELETE | 删除账号 |
| `/api/admin/credentials/:id/balance` | GET | 查询余额 |

---

## 模型映射

| 客户端请求模型 | Kiro 实际模型 | 说明 |
|----------------|---------------|------|
| `claude-sonnet-4-0` | `anthropic.claude-sonnet-4-0-v3:0` | Claude Sonnet 4 |
| `claude-sonnet-4-5` | `anthropic.claude-sonnet-4-5-v2:0` | Claude Sonnet 4.5 |
| `claude-sonnet-4-6` | `anthropic.claude-sonnet-4-6-v2:0` | Claude Sonnet 4.6 |
| `claude-opus-4` | `anthropic.claude-opus-4-0-v1:0` | Claude Opus 4 |
| `claude-opus-4-5` | `anthropic.claude-opus-4-5-v1:0` | Claude Opus 4.5 |
| `claude-opus-4-6` | `anthropic.claude-opus-4-6-v1:0` | Claude Opus 4.6 |
| `claude-opus-4-7` | `anthropic.claude-opus-4-7-v1:0` | Claude Opus 4.7 |
| `claude-opus-4-8` | `anthropic.claude-opus-4-8-v1:0` | Claude Opus 4.8 |
| `claude-haiku-4-5` | `anthropic.claude-haiku-4-5-v1:0` | Claude Haiku 4.5 |
| `deepseek-v3-2` | `deepseek.deepseek-v3-2:0` | DeepSeek 3.2 |
| `glm-5` | `glm.glm-5:0` | GLM-5 |
| `minimax-m2-1` | `minimax.minimax-m2-1:0` | MiniMax M2.1 |
| `minimax-m2-5` | `minimax.minimax-m2-5:0` | MiniMax M2.5 |
| `qwen3-coder` | `qwen.qwen3-coder:0` | Qwen3-Coder |
| `gpt-5-6-sol` | `openai.gpt-5-6-sol:0` | GPT-5.6 Sol |
| `gpt-5-6-terra` | `openai.gpt-5-6-terra:0` | GPT-5.6 Terra |
| `gpt-5-6-luna` | `openai.gpt-5-6-luna:0` | GPT-5.6 Luna |

---

## Admin 管理面板

### 访问管理面板

浏览器访问：`http://localhost:5678/admin`

使用 `config.json` 中配置的 `adminApiKey` 登录。

### 主要功能

- **账号管理**：添加、编辑、删除 Kiro 账号
- **余额查询**：查看每个账号的剩余额度
- **Token 刷新**：手动刷新账号 Token
- **日志查看**：查看服务运行日志
- **配置管理**：在线编辑服务配置

---

## 常见问题

**Q：启动后提示"已加载 0 个账号配置"**

需要创建 `data/credentials.json` 并添加至少一个账号配置，参考[获取 Kiro 账号](#获取-kiro-账号)章节。

**Q：请求返回 `INVALID_MODEL_ID`**

> ⚠️ **【重要】** 国内 IP 无法直接访问 Claude 模型。必须在 `data/config.json` 中配置 `proxyUrl`（如 `"proxyUrl": "http://host.docker.internal:7890"`），或使用境外服务器。这是国内用户最常见的问题。

**Q：使用 GPT-5.6 系列模型（sol/terra/luna）时，thinking 模式、output effort 或 max_tokens 设定似乎没有生效**

GPT-5.6 系列的 Kiro 后端 schema 不支持 `additionalModelRequestFields`（涵盖 thinking / output_config effort / max_tokens 三个子字段），与 Claude 4.5 代际（Sonnet 4.5 / Opus 4.5 / Haiku 4.5）一样会被整体跳过，属已知限制，非本项目 bug。

**Q：请求返回 401 Unauthorized**

客户端使用的 API Key 与 `config.json` 中的 `apiKey` 不一致，检查并对齐。

**Q：Token 刷新失败 / 请求报错**

尝试将 `config.json` 中的 `tlsBackend` 改为 `native-tls` 后重启服务。

**Q：通过管理面板导入账号时报错 `Cannot read properties of undefined (reading 'digest')`**

此问题已在 v2.7.3 修复：`crypto.subtle` 加密 API 只在 HTTPS 或 localhost 环境下可用，公网 IP + HTTP 访问会触发此错误，v2.7.3 起自动降级为纯 JS 实现，无需再配置 HTTPS。若仍报错，请升级到最新版本。

**Q：企业版 IdC 账号请求返回 502，日志显示 `profileArn is required for this request`**

企业版（Enterprise）IdC 账号调用 Q 端点强制要求 `profileArn`，但 IdC Token 刷新接口不会返回该字段，需要手动填写。管理面板「添加账号 / 编辑账号」对话框中已提供 **Profile ARN** 输入框，填入形如 `arn:aws:codewhisperer:<region>:<account-id>:profile/<profile-id>` 的值即可。`profileArn` 可从 Kiro IDE 本地缓存或 `ListAvailableProfiles` 获取，其所在 region 需与账号的 `apiRegion` 保持一致。Social 账号一般无需填写。

**Q：子 API Key 消费额度能按真实 Kiro credits 计量吗（而不是估算的美元）**

能。创建/编辑子 API Key 时，额度单位可选「美元估算」或「真实 Credits」（limitUnit：usd/credits）。选择 credits 时，额度按 usage 记录中的真实 credits_used 累加计量（旧记录无 credits_used 字段时按 estimated_cost × k_ref 回退估算）。默认为 usd，向后兼容现有配置。

**Q：容器无法访问宿主机代理**

确保 `docker-compose.yml` 中包含以下配置：

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

然后在 `config.json` 中使用 `http://host.docker.internal:端口号` 作为代理地址。

**Q：如何查看容器日志？**

```bash
docker compose logs -f kiro2cc-proxy
```

**Q：如何进入容器调试？**

```bash
docker compose exec kiro2cc-proxy sh
```

---

## 注意事项

1. `credentials.json` 包含敏感 Token，不要提交到版本控制，不要分享给他人
2. 服务会自动刷新过期 Token，无需手动干预
3. 多账号模式下 Token 刷新后自动回写到文件
4. 国内用户必须配置代理才能访问 Claude 模型
5. Docker 部署时，配置文件位于 `./data` 目录，会自动挂载到容器内

---

## 项目结构

```
kiro-proxy/
├── src/                    # Rust 源码
├── admin-ui/               # 管理面板前端
├── user-ui/                # 用户面板前端
├── data/                   # Docker 配置目录
│   ├── config.json         # 服务配置
│   └── credentials.json    # 账号配置
├── config.example.json     # 配置示例
├── docker-compose.yml      # Docker Compose 配置
├── Dockerfile              # Docker 镜像构建文件
└── README.md               # 本文档
```

---

## License

MIT

## 致谢

本项目基于 [kiro.rs](https://github.com/hank9999/kiro.rs) 二次开发，感谢原作者的开源贡献。
