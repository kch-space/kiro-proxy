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
- **账号导入导出**：KAM 兼容格式的批量导入与一键导出，方便跨服务器迁移
- **账号级代理**：支持为每个账号单独配置 HTTP/SOCKS5 代理

---

## 快速开始

### 前置要求

- 安装 Docker 和 Docker Compose
- 一个或多个 Kiro 账号的 `refreshToken`

### 选择部署方式

本项目支持两种部署方式，配置文件和使用方式完全一致，区别只在镜像从哪来：

| | 方式 A：拉取预构建镜像 | 方式 B：从源码构建 |
|---|---|---|
| 操作 | 一步：`up -d` | 两步：先 `build` 再 `up -d` |
| 耗时 | 10-60 秒 | 7-15 分钟 |
| 额外要求 | 无 | Git、内存 ≥ 2GB、可用磁盘 ≥ 10GB |
| 能否用自己改的代码 | 不能 | 能 |
| 架构适配 | 由镜像仓库自动匹配 | 构建机即运行机，天然一致 |
| 构建产物 | 无 | 本地镜像 `kiro-proxy:latest`，可推到自己的仓库复用 |
| 适合 | 开箱即用、快速上线 | 改过代码、或想自己掌控构建产物 |

> 💡 拿不定就先用**方式 A**，之后想改代码再切到方式 B，两者可以随时互换，数据不受影响。
>
> 💡 **多台服务器**：用方式 B 在一台机器上构建好，把镜像推到自己的镜像仓库，其余机器改用方式 A 拉取即可，不必每台都编译一遍。见「构建说明 → 发布到镜像仓库」。
>
> ⚠️ **注意**：方式 B 的构建对内存要求较高，Rust 编译阶段在 1GB 内存的小规格服务器上可能被 OOM Kill。这种情况可以在本机构建好镜像再传到服务器，见「构建说明 → 低配服务器：本地构建后传输」。

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

按前面选定的方式二选一执行。

**方式 A：拉取预构建镜像**

```bash
docker compose up -d
```

> 首次运行会自动拉取镜像（约 37MB），需要 10-60 秒；后续启动直接使用本地缓存，只需几秒钟。

**方式 B：从源码构建**

分两步：**先构建镜像，再用构建好的镜像启动**。叠加 `docker-compose.build.yml` 即可把镜像来源换成本地源码。

第一步，构建镜像：

```bash
docker compose -f docker-compose.yml -f docker-compose.build.yml build
```

首次构建需要 7-15 分钟（前端编译 2-4 分钟 + Rust 编译 5-10 分钟），产物是本地镜像 `kiro-proxy:latest`。

第二步，启动服务：

```bash
docker compose -f docker-compose.yml -f docker-compose.build.yml up -d
```

这一步只需几秒，直接使用上一步构建好的镜像，不会重新编译。

> 💡 嫌命令太长，可以在项目根目录创建 `.env` 写入一行，之后 `docker compose build` / `docker compose up -d` 就够了：
>
> ```bash
> echo "COMPOSE_FILE=docker-compose.yml:docker-compose.build.yml" > .env
> ```
>
> 后续改了代码，重复这两步即可（构建有缓存，通常比首次快很多）。

> 💡 构建产物是一个独立的本地镜像，可以打 tag 推到自己的镜像仓库，之后这台机器和其他机器都能改用方式 A 直接拉取，不必每台都编译一遍。见「构建说明 → 发布到镜像仓库」。

**5. 访问管理面板**

打开浏览器访问 `http://localhost:5678/admin`，使用 `config.json` 中配置的 `adminApiKey` 登录，然后通过「批量导入」或「Kiro Account Manager 导入」添加 Kiro 账号（见「配置说明 → 添加账号」）。

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
3. 按账号来源选择导入入口：
   - **Kiro Account Manager 导入**：来自 KAM 或本项目「导出账号」的 JSON，可直接粘贴或把 `.json` 文件拖进对话框，导入前会显示识别到的账号数量
   - **批量导入**：自己整理的 JSON，格式较宽松（账号数组、`{ "accounts": [...] }`、单个对象都支持），单个账号也从这里粘贴
4. 点击「开始导入并验活」

两个入口都会逐个查重、添加并验活，验活失败的账号会自动回滚删除，导入过程有进度和逐条结果。

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

### 导出账号 / 迁移到其他服务器

管理面板支持把账号导出成 JSON 文件，用于迁移到另一台服务器，无需手工去翻容器里的 `data/credentials.json`。

1. 访问 `http://localhost:5678/admin` 并用 `adminApiKey` 登录
2. 点击工具栏的「导出账号」按钮，浏览器会下载 `kiro-accounts-<时间戳>.json`
   - 未勾选任何账号时导出全部；勾选后按钮变为「导出选中 (N)」，只导出勾选的账号
3. 在新服务器的管理面板用「批量导入」或「KAM 导入」读入该文件

导出格式兼容 KAM（Kiro Account Manager），并额外保留了本项目特有的 `apiRegion`、`priority`、账号级代理配置，因此迁移后这些设置不会丢失。文件不包含 `accessToken`，目标服务器会用 `refreshToken` 自动重新获取。

> ⚠️ **安全提示**：导出文件内含**明文** refreshToken，等同于账号密码。请通过可信渠道传输，导入完成后及时删除，切勿提交到版本控制或分享给他人。

> 📌 **已知限制**：账号的**禁用状态不会随导入恢复**。导出文件中的 `disabled` 字段仅作记录，导入后所有账号均为启用态，如有需要请在新服务器上手动禁用。

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

### 更新服务（重新构建镜像）

已部署在服务器上的服务，升级时**数据不会丢失**：`docker-compose.yml` 把 `./data` 挂载进容器（`./data:/app/config`），账号、API Key、用量数据都在宿主机上，容器重建不影响它们。

#### 更新前备份（建议）

```bash
# 备份整个数据目录
cp -r data "data.bak.$(date +%Y%m%d-%H%M%S)"
```

也可以先打开管理面板，用「导出账号」按钮把账号导出成 JSON 留存一份（见「导出账号 / 迁移到其他服务器」）。

建议同时给当前镜像打一个备份 tag，方便回滚（镜像名按你的部署方式选）：

```bash
# 方式 A
docker tag kch9231/kiro-proxy:latest "kch9231/kiro-proxy:backup-$(date +%Y%m%d)"

# 方式 B
docker tag kiro-proxy:latest "kiro-proxy:backup-$(date +%Y%m%d)"
```

#### 方式 A：拉取新镜像

```bash
cd /path/to/kiro-proxy
git pull                 # 同步 docker-compose.yml 等配置的变更
docker compose pull      # 拉取最新镜像
docker compose up -d     # 用新镜像重建容器
```

#### 方式 B：拉取新代码并重新构建

```bash
cd /path/to/kiro-proxy
git pull
docker compose -f docker-compose.yml -f docker-compose.build.yml build   # 重新构建镜像
docker compose -f docker-compose.yml -f docker-compose.build.yml up -d   # 用新镜像重建容器
```

（若已在 `.env` 中配置 `COMPOSE_FILE`，直接 `docker compose build && docker compose up -d` 即可。）

要点：

- **`build` 这一步不能省**。直接 `up -d` 的话，compose 发现本地已有 `kiro-proxy:latest` 就直接复用，新代码不会生效
- 构建期间**旧容器仍在提供服务**，只有到第二步 `up -d` 才会停旧容器、起新容器；构建失败则服务完全不受影响
- 服务器配置不足时改用本地构建再传输，见「构建说明 → 低配服务器：本地构建后传输」

#### 更新后验证

两种方式通用：

```bash
docker compose ps                                          # 容器状态应为 running
docker compose exec kiro2cc-proxy ./kiro2cc-proxy --version # 确认版本号已更新
docker compose logs --tail=30                              # 检查启动日志和账号加载数量
```

启动日志里应能看到「已加载 N 个账号配置」，N 与更新前一致。

#### 回滚

前提是升级前打过备份 tag（见上文）。把备份 tag 重新指回 `latest`，然后重建容器：

```bash
# 方式 A
docker tag kch9231/kiro-proxy:backup-20260905 kch9231/kiro-proxy:latest
docker compose up -d

# 方式 B
docker tag kiro-proxy:backup-20260905 kiro-proxy:latest
docker compose up -d
```

方式 B 若要回滚到某个历史版本的代码，用 `git checkout <可用的 commit>` 后重新执行 `build` + `up -d`。

#### 清理旧镜像和构建缓存

反复构建会留下大量无 tag 的旧镜像层和构建缓存，很占磁盘：

```bash
docker image prune -f      # 清理无 tag 的悬空镜像，安全
docker builder prune -f    # 清理构建缓存（源码构建的缓存可达数 GB）
```

> ⚠️ 不要用 `docker system prune -a`，它会删掉所有当前未被容器使用的镜像，包括你留作回滚的备份 tag。
>
> ⚠️ 清理构建缓存后，下一次 `build` 会退化成完整重新编译（7-15 分钟）。磁盘不紧张时可以保留缓存。

### 停止服务

```bash
docker compose down
```

---

## 构建说明

本章节针对**方式 B（从源码构建）**，说明构建原理和几种特殊场景。常规部署和更新流程见「快速开始」和「更新服务」章节。

源码构建由 `docker-compose.build.yml` 提供，它只覆盖两个字段——加上 `build: .`、把镜像 tag 换成本地的 `kiro-proxy:latest`，其余配置全部沿用 `docker-compose.yml`。

> 💡 **在部署机上构建的好处**：构建机与运行机是同一台，CPU 架构天然一致。相对地，如果在 Apple Silicon 的 Mac（arm64）上构建镜像再传到 x86_64 服务器，容器会因架构不符直接报 `exec format error` 无法启动，必须显式交叉构建才行（见下文「注意 CPU 架构」）。

### 构建过程

`Dockerfile` 为三阶段构建：

1. **前端阶段**（`node:22-alpine`）：编译 admin-ui 和 user-ui，产物为静态文件
2. **后端阶段**（`rust:1-alpine`）：前端产物通过 `rust-embed` 编译进二进制，`cargo build --release`
3. **运行阶段**（`alpine:3.21`）：只包含最终二进制和 CA 证书，镜像约 37MB

因为前端产物是编译进二进制的，**改动前端代码同样需要重新构建整个镜像**，不能只替换静态文件。

### 低配服务器：本地构建后传输

服务器内存不足（< 2GB）或不想在生产机上跑编译时，在本机构建好再传过去。

```bash
# 本机：构建并导出压缩包
docker build -t kiro-proxy:latest .
docker save kiro-proxy:latest | gzip > kiro-proxy.tar.gz
scp kiro-proxy.tar.gz user@server:/tmp/

# 服务器：导入镜像
gunzip -c /tmp/kiro-proxy.tar.gz | docker load
rm /tmp/kiro-proxy.tar.gz
```

导入后要让 compose 用这个本地镜像，而不是去仓库拉取——把服务器上 `docker-compose.yml` 的 `image` 改成导入时的标签：

```yaml
services:
  kiro2cc-proxy:
    image: kiro-proxy:latest # 原为 kch9231/kiro-proxy:latest
```

然后启动（不要叠加 `docker-compose.build.yml`，否则会在服务器上重新编译）：

```bash
cd /path/to/kiro-proxy && docker compose up -d
```

服务器上仍需要有本仓库的 `docker-compose.yml` 和 `data/` 目录，但不需要完整源码。后续更新重复「本机构建 → 传输 → `docker load` → `docker compose up -d`」即可。

> ⚠️ **注意 CPU 架构**：本机与服务器架构不一致时（例如本机是 Apple Silicon arm64、服务器是 x86_64），必须指定目标平台构建，否则镜像在服务器上无法运行：
>
> ```bash
> docker build --platform linux/amd64 -t kiro-proxy:latest .
> ```
>
> 跨架构构建走 QEMU 模拟，耗时通常是原生构建的 3-5 倍。

### 发布到镜像仓库（可选）

管理多台服务器时，可以构建一次推到自己的仓库，其他机器直接按方式 A 拉取，省去每台重复编译。

**情况一：已经用方式 B 构建过，且构建机与目标服务器架构相同**

方式 B 的产物就是一个完整镜像，打个 tag 直接推，不需要重新构建：

```bash
docker login
docker tag kiro-proxy:latest your-username/kiro-proxy:latest
docker push your-username/kiro-proxy:latest
```

例如在 x86_64 服务器上用方式 B 构建，推上去的就是 amd64 镜像，其他 x86_64 服务器可以直接拉。

**情况二：构建机与目标服务器架构不同**

典型场景是在 Apple Silicon 的 Mac（arm64）上构建、推给 x86_64 服务器。这时不能沿用上面的 tag + push——`docker build` 产出的是构建机自己的架构，服务器拉下来会直接报 `exec format error`。必须用 `buildx` 显式指定平台重新构建：

```bash
docker login

# 只部署到 x86_64 服务器
docker buildx build --platform linux/amd64 \
  -t your-username/kiro-proxy:latest --push .

# 或一次推送多架构镜像，服务器拉取时自动选择匹配的那个
docker buildx build --platform linux/amd64,linux/arm64 \
  -t your-username/kiro-proxy:latest --push .
```

> 💡 `buildx build --push` 会直接推送，不会在本地留下镜像；想同时保留本地副本可加 `--load`（仅限单平台）。
>
> ⚠️ 跨架构构建走 QEMU 模拟，耗时通常是原生构建的 3-5 倍。

**推送后切回方式 A**

把服务器上 `docker-compose.yml` 的 `image` 改为自己的镜像地址：

```yaml
services:
  kiro2cc-proxy:
    image: your-username/kiro-proxy:latest # 原为 kch9231/kiro-proxy:latest
```

之后这台机器按方式 A 部署和更新即可（不要再叠加 `docker-compose.build.yml`，若之前建过 `.env` 记得删掉其中的 `COMPOSE_FILE`）：

```bash
docker compose pull && docker compose up -d
```

以后代码有改动，只需在一台机器上构建并推送新镜像，其余机器执行上面这条命令就能更新。

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

在管理面板的「编辑账号」中填写 **Profile ARN**，格式如：`arn:aws:codewhisperer:<region>:<account-id>:profile/<profile-id>`；批量导入时也可以在 JSON 里带上 `profileArn` 字段。

**Q：拉取镜像失败（方式 A）**

从 Docker Hub 拉取失败时可以：

1. 为 Docker 配置镜像加速器
2. 或改用方式 B 从源码构建（见「快速开始 → 选择部署方式」）

**Q：构建过程中被中断，日志出现 `signal: 9` 或 `Killed`（方式 B）**

内存不足被系统 OOM Kill，Rust 编译阶段最容易触发。可选方案：

1. 临时加 swap（1GB 内存的服务器建议加 2GB）：
   ```bash
   sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
   sudo mkswap /swapfile && sudo swapon /swapfile
   ```
2. 改为在本机构建后传输镜像，见「构建说明 → 低配服务器：本地构建后传输」
3. 或直接改用方式 A 拉取预构建镜像

**Q：构建时拉取基础镜像或依赖失败（方式 B）**

构建需要访问 Docker Hub、npm 和 crates.io。国内网络可以：

1. 为 Docker 配置镜像加速器（解决基础镜像拉取）
2. 给 Docker daemon 配置代理（解决 npm / crates.io 下载）

**Q：改了代码，重启后没生效**

先确认用的是方式 B——方式 A 拉的是预构建镜像，本地代码改动不会进入镜像。

方式 B 下，`docker compose up -d` 只会用现有的 `kiro-proxy:latest` 镜像，不会自己重新构建。必须先跑 `build`：

```bash
docker compose -f docker-compose.yml -f docker-compose.build.yml build
docker compose -f docker-compose.yml -f docker-compose.build.yml up -d
```

前端代码同样如此——前端产物是编译进 Rust 二进制的，改动前端也要重新构建整个镜像。

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

本项目基于 [TsinHzl/kiro2cc-proxy](https://github.com/TsinHzl/kiro2cc-proxy) 二次开发，感谢原作者的开源贡献。
