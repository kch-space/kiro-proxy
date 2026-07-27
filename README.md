# Kiro-Proxy

Anthropic Claude API 兼容代理服务，将 Claude API 请求转发到 Kiro。

## 免责声明

本项目仅供研究使用，Use at your own risk。使用本项目所导致的任何后果由使用人承担，与本项目无关。本项目与 AWS/KIRO/Anthropic/Claude 等官方无关，不代表官方立场。

## 功能特性

- **API 兼容**：完整支持 Anthropic Claude API 格式
- **流式响应**：支持 SSE 流式输出
- **Token 自动刷新**：自动管理和刷新 OAuth Token
- **多账号支持**：支持配置多个账号，按优先级自动故障转移
- **负载均衡**：支持 `priority`（按优先级）和 `balanced`（均衡分配）两种模式
- **智能重试**：单账号最多重试 3 次，单请求最多重试 9 次
- **Web 管理**：可选的 Web 管理界面，支持账号管理、余额查询等
- **账号级代理**：支持为每个账号单独配置 HTTP/SOCKS5 代理

---

## 快速开始

### 前置要求

- 安装 Docker 和 Docker Compose
- 一个或多个 Kiro 账号的 `refreshToken`

### 部署步骤

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

创建 `data/config.json`（必填项已标注 ⚠️）：

```bash
cat > data/config.json << 'EOL'
{
  "host": "0.0.0.0",
  "port": 5678,
  "apiKey": "sk-kiro2cc-proxy-qazWSXedcRFV123456",
  "tlsBackend": "rustls",
  "region": "us-east-1",
  "adminApiKey": "sk-admin-your-secret-key"
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

> 💡 **首次启动说明**：
> - 首次运行会自动拉取 Docker 镜像（约 10MB），需要 **10-60 秒**（取决于网络速度）
> - 后续启动会直接使用已缓存的镜像，只需几秒钟

**5. 访问管理面板**

打开浏览器访问 `http://localhost:5678/admin`，使用 `config.json` 中配置的 `adminApiKey` 登录，然后添加 Kiro 账号。

---

## 配置说明

### config.json 配置项

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `apiKey` | string | ⚠️ 是 | - | 客户端访问服务的 API Key |
| `host` | string | 否 | `127.0.0.1` | 监听地址，Docker 建议 `0.0.0.0` |
| `port` | number | 否 | `5678` | 监听端口 |
| `adminApiKey` | string | 否 | - | 管理面板的 API Key（不填则无法访问管理面板） |
| `proxyUrl` | string | 否 | - | 全局代理地址（如 `http://host.docker.internal:7890`） |
| `balanceMode` | string | 否 | `priority` | 负载均衡模式：`priority` 或 `balanced` |
| `tlsBackend` | string | 否 | `rustls` | TLS 后端：`rustls` 或 `native-tls` |

> ⚠️ **国内用户必读**：必须配置 `proxyUrl`，否则 Claude 模型请求会返回 `INVALID_MODEL_ID` 错误。
>
> **在 Docker 中使用代理的说明**：
> - `host.docker.internal` 是 Docker 提供的特殊域名，指向宿主机（你的电脑）
> - 容器内的 `127.0.0.1` 只能访问容器自己，无法访问宿主机的代理软件
> - 配置示例：`"proxyUrl": "http://host.docker.internal:7890"`（7890 改为你的代理端口）

### credentials.json 配置项

每个账号对象包含以下字段：

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `name` | string | ⚠️ 是 | - | 账号名称（便于识别） |
| `refreshToken` | string | ⚠️ 是 | - | Kiro 账号的 refresh token |
| `apiRegion` | string | 否 | `us-east-1` | API 区域 |
| `priority` | number | 否 | `1` | 优先级（数字越小越优先） |
| `proxyUrl` | string | 否 | - | 该账号专用代理（覆盖全局代理） |
| `profileArn` | string | 否 | - | 企业版 IdC 账号需要填写 |

### 添加账号

**方式一：通过管理面板（推荐）**

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

## 接入 Claude Code

### 配置步骤

在 Claude Code 设置中：

1. **API Provider**: 选择 `Anthropic`
2. **API Key**: 填入你在 `config.json` 中设置的 `apiKey`
3. **Base URL**: 填入代理服务地址
   - 本地部署：`http://localhost:5678`
   - 远程部署：`http://your-server-ip:5678`

配置完成后，Claude Code 会通过代理服务访问 Kiro 账号的模型。

---

## 常用命令

### 查看日志

```bash
docker compose logs -f
```

### 重启服务

```bash
docker compose restart
```

### 更新服务

**使用官方镜像（推荐）**

拉取最新代码和镜像：

```bash
git pull
docker compose pull
docker compose up -d
```

**使用自定义构建**

如果你修改了代码，需要重新构建镜像（参见"自定义构建"章节）：

```bash
docker compose up -d --build
```

### 停止服务

```bash
docker compose down
```

---

## 自定义构建

如果你需要修改代码并自行构建镜像，请按以下步骤操作：

### 构建步骤

**1. 修改 docker-compose.yml**

在 `docker-compose.yml` 中添加 `build` 配置：

```yaml
services:
  kiro2cc-proxy:
    build: .
    image: kiro2cc-proxy:latest
    container_name: kiro2cc-proxy
    extra_hosts:
      - "host.docker.internal:host-gateway"
    ports:
      - "0.0.0.0:5678:5678"
    volumes:
      - ./data:/app/config
    restart: unless-stopped
```

**2. 构建并启动**

```bash
docker compose up -d --build
```

> ⚠️ **构建时间说明**：
> - 首次构建需要 **7-15 分钟**（取决于网络速度和 CPU 性能）
> - 构建过程包括：前端编译（2-4 分钟）+ Rust 编译（5-10 分钟）
> - 后续启动会直接使用已构建的镜像，只需几秒钟

**3. 更新自定义构建**

修改代码后重新构建：

```bash
docker compose up -d --build
```

**4. 发布自定义镜像到 Docker Hub（可选）**

如果你想发布自己的镜像到 Docker Hub：

```bash
# 构建镜像
docker compose build

# 登录 Docker Hub
docker login

# 标记镜像（替换为你的 Docker Hub 用户名）
docker tag kiro2cc-proxy:latest your-dockerhub-username/kiro-proxy:latest

# 推送镜像
docker push your-dockerhub-username/kiro-proxy:latest

# 更新 docker-compose.yml 中的镜像名称
# image: your-dockerhub-username/kiro-proxy:latest
```

之后可以在其他机器上直接使用你发布的镜像。

---

## 常见问题

**Q：启动后提示"已加载 0 个账号配置"**

通过管理面板或手动编辑 `data/credentials.json` 添加至少一个账号。

**Q：请求返回 `INVALID_MODEL_ID`**

国内 IP 无法直接访问 Claude 模型，必须在 `data/config.json` 中配置 `proxyUrl`。Docker 中使用 `http://host.docker.internal:端口号`。

**Q：请求返回 401 Unauthorized**

客户端使用的 API Key 与 `config.json` 中的 `apiKey` 不一致。

**Q：Token 刷新失败**

尝试将 `config.json` 中的 `tlsBackend` 改为 `native-tls` 后重启服务。

**Q：容器无法访问宿主机代理**

确保 `docker-compose.yml` 中包含：

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

这个配置让容器能访问宿主机上的服务（如代理软件）。如果没有这行，容器内的 `host.docker.internal` 无法解析。

**Q：企业版 IdC 账号请求返回 502**

在管理面板「添加账号 / 编辑账号」中填写 **Profile ARN**，格式如：`arn:aws:codewhisperer:<region>:<account-id>:profile/<profile-id>`

**Q：拉取 Docker 镜像失败**

如果从 Docker Hub 拉取镜像失败，可以尝试：
1. 配置 Docker 镜像加速器
2. 或使用自定义构建方式（参见"自定义构建"章节）

---

## 注意事项

1. `credentials.json` 包含敏感 Token，不要提交到版本控制，不要分享给他人
2. 服务会自动刷新过期 Token，无需手动干预
3. 国内用户必须配置代理才能访问 Claude 模型
4. Docker 部署时，配置文件位于 `./data` 目录，会自动挂载到容器内

---

## License

MIT

## 致谢

本项目基于 [kiro.rs](https://github.com/hank9999/kiro.rs) 二次开发，感谢原作者的开源贡献。
