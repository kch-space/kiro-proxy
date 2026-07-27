# kiro2cc-proxy

[![Tests](https://github.com/TsinHzl/kiro2cc-proxy/actions/workflows/test.yaml/badge.svg)](https://github.com/TsinHzl/kiro2cc-proxy/actions/workflows/test.yaml)
[![codecov](https://codecov.io/gh/TsinHzl/kiro2cc-proxy/graph/badge.svg)](https://codecov.io/gh/TsinHzl/kiro2cc-proxy)

A Rust-based Anthropic Claude API-compatible proxy that converts Anthropic API requests into Kiro API requests.

> **✅ Supported Models: Claude Sonnet 5 / Claude Sonnet 4.5 / Claude Sonnet 4.6 / Claude Opus 4.5 / Claude Opus 4.6 / Claude Opus 4.7 / Claude Opus 4.8 / Claude Haiku 4.5 / DeepSeek 3.2 / GLM-5 / MiniMax M2.1 / MiniMax M2.5 / Qwen3-Coder / GPT-5.6 Sol / GPT-5.6 Terra / GPT-5.6 Luna**

[中文](README.md) | English

## Disclaimer

This project is for research purposes only. Use at your own risk. Any consequences arising from the use of this project are solely the responsibility of the user. This project is not affiliated with AWS, KIRO, Anthropic, or Claude in any official capacity.

## Features

- **Anthropic API Compatible**: Full support for the Anthropic Claude API format
- **Streaming Responses**: SSE (Server-Sent Events) streaming support
- **Auto Token Refresh**: Automatically manages and refreshes OAuth tokens
- **Multi-Account Support**: Configure multiple accounts with automatic priority-based failover
- **Load Balancing**: `priority` (by priority) and `balanced` (round-robin) modes
- **Smart Retry**: Up to 3 retries per account, up to 9 retries per request
- **Thinking Mode**: Supports Claude's extended thinking feature
- **Tool Use**: Full support for function calling / tool use
- **WebSearch**: Built-in WebSearch tool conversion logic
- **Admin Panel**: Optional web management UI for account management, balance queries, etc.
- **Per-Account Proxy**: Configure HTTP/SOCKS5 proxy per account

---

## Table of Contents

- [Quick Start](#quick-start)
- [Docker Deployment](#docker-deployment)
- [Getting Kiro Accounts](#getting-kiro-accounts)
- [Configuration Reference](#configuration-reference)
- [Claude Code Integration](#claude-code-integration)
- [API Endpoints](#api-endpoints)
- [Model Mapping](#model-mapping)
- [Admin Panel](#admin-panel)
- [FAQ](#faq)
- [Notes](#notes)

---

## Quick Start

**What is this project?**

kiro2cc-proxy is a proxy service. It forwards standard Anthropic Claude API requests to Kiro (AWS's AI coding tool), allowing you to use Claude Code with models from your Kiro account.

> In short: it proxies the models on your logged-in Kiro account to Claude Code. Without it, you can only use those models inside Kiro IDE or Kiro CLI.

**Prerequisites:**

1. A Kiro account (register at [kiro.dev](https://kiro.dev), supports Social login)
2. Account credentials exported from Kiro IDE or account manager (`refreshToken` etc.)
3. > ⚠️ **[CRITICAL] Users in mainland China**: A local HTTP/SOCKS5 proxy (Clash/V2Ray etc.) is mandatory. Without it, all Claude model requests will return `INVALID_MODEL_ID` and the service will be unusable.

**Overall flow:**

```
Install Docker → Deploy service → Add accounts → Configure client
```

---

## Docker Deployment

### Prerequisites

- Docker and Docker Compose installed
- One or more Kiro account `refreshToken`s

### Quick Deploy

**1. Clone the repository**

```bash
git clone https://github.com/kch-space/kiro-proxy.git
cd kiro-proxy
```

**2. Create data directory**

```bash
mkdir -p data
```

**3. Create configuration files**

Create `data/config.json`:

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

Create `data/credentials.json` (initially empty array):

```bash
echo "[]" > data/credentials.json
```

**4. Start the service**

```bash
docker compose up -d
```

**5. View logs**

```bash
docker compose logs -f
```

**6. Access admin panel**

Open browser and visit `http://localhost:5678/admin`, login with the `adminApiKey` configured in `config.json`.

### Configuration

**Environment Variables (Optional)**

You can also configure via environment variables in `docker-compose.yml`:

```yaml
services:
  kiro2cc-proxy:
    environment:
      - API_KEY=your-api-key-here
      - ADMIN_API_KEY=your-admin-key-here
      - HOST=0.0.0.0
      - PORT=5678
```

**Proxy Configuration for Users in China**

> ⚠️ **Important**: Users in mainland China must configure a proxy to access Claude models

Add `proxyUrl` in `data/config.json`:

```json
{
  "proxyUrl": "http://host.docker.internal:7890"
}
```

`host.docker.internal` automatically resolves to the host machine IP (requires Docker 18.03+).

### Update Service

```bash
cd kiro-proxy
git pull
docker compose pull
docker compose down && docker compose up -d
```

### Stop Service

```bash
docker compose down
```

### Complete Cleanup

```bash
docker compose down -v
rm -rf data
```

---

## Getting Kiro Accounts

### Method 1: Via Kiro IDE (Recommended)

1. Install and login to [Kiro IDE](https://kiro.dev)
2. Open command palette (macOS: `Cmd+Shift+P` / Windows: `Ctrl+Shift+P`)
3. Type `Kiro: Export Credentials`
4. Select account, JSON format credentials will be copied to clipboard

### Method 2: Via Browser DevTools

1. Visit [kiro.dev](https://kiro.dev) and login
2. Open browser DevTools (F12)
3. Switch to Console tab
4. Execute the following script:

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
  alert('Credentials copied to clipboard');
})();
```

### Account Configuration Format

```json
{
  "name": "Account Name",
  "refreshToken": "refresh_token_value",
  "apiRegion": "us-east-1",
  "priority": 1,
  "proxyUrl": ""
}
```

**Field Descriptions:**

- `name`: Account name (for identification)
- `refreshToken`: Kiro account refresh token (required)
- `apiRegion`: API region, defaults to `us-east-1` (optional)
- `priority`: Priority, lower number = higher priority (optional, defaults to 1)
- `proxyUrl`: Proxy URL for this specific account (optional)

### Adding Accounts

**Method 1: Via Admin Panel**

1. Visit `http://localhost:5678/admin`
2. Login with `adminApiKey`
3. Click "Add Account" button
4. Fill in account information and save

**Method 2: Manually Edit Config File**

Edit `data/credentials.json`:

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

After editing, restart the service:

```bash
docker compose restart
```

---

## Configuration Reference

### config.json Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `apiKey` | string | Yes | - | API Key for client access |
| `host` | string | No | `127.0.0.1` | Listen address, recommend `0.0.0.0` for Docker |
| `port` | number | No | `5678` | Listen port |
| `adminApiKey` | string | No | - | Admin panel API Key |
| `proxyUrl` | string | No | - | Global proxy URL (e.g., `http://127.0.0.1:7890`) |
| `balanceMode` | string | No | `priority` | Load balancing mode: `priority` or `balanced` |
| `tlsBackend` | string | No | `rustls` | TLS backend: `rustls` or `native-tls` |

### credentials.json Fields

Each account object contains the following fields:

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `name` | string | Yes | - | Account name |
| `refreshToken` | string | Yes | - | Kiro account refresh token |
| `apiRegion` | string | No | `us-east-1` | API region |
| `priority` | number | No | `1` | Priority (lower = higher priority) |
| `proxyUrl` | string | No | - | Account-specific proxy |
| `profileArn` | string | No | - | Required for Enterprise IdC accounts |

### Load Balancing Modes

**priority mode (default)**

Sorts by `priority` field, uses higher priority accounts first. When current account fails, automatically switches to next priority account.

**balanced mode**

Round-robin across all available accounts for load balancing. Suitable for scenarios with multiple accounts.

---

## Claude Code Integration

### Step 1: Install Claude Code

Install from [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=Anthropic.claude-code) or your IDE plugin marketplace.

### Step 2: Configure API

In Claude Code settings:

1. **API Provider**: Select `Anthropic`
2. **API Key**: Enter the `apiKey` you set in `config.json`
3. **Base URL**: Enter proxy service address
   - Local deployment: `http://localhost:5678`
   - Remote deployment: `http://your-server-ip:5678`

### Step 3: Start Using

After configuration, Claude Code will access Kiro account models through the proxy service.

---

## API Endpoints

### Core Endpoints

| Endpoint | Description |
|----------|-------------|
| `POST /v1/messages` | Anthropic-compatible messages API |
| `POST /v1/chat/completions` | OpenAI-compatible chat API (experimental) |

### Admin Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/admin/credentials` | GET | Get all accounts |
| `/api/admin/credentials` | POST | Add account |
| `/api/admin/credentials/:id` | DELETE | Delete account |
| `/api/admin/credentials/:id/balance` | GET | Query balance |

---

## Model Mapping

| Client Request Model | Kiro Actual Model | Description |
|----------------------|-------------------|-------------|
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

## Admin Panel

### Accessing Admin Panel

Visit in browser: `http://localhost:5678/admin`

Login with the `adminApiKey` configured in `config.json`.

### Key Features

- **Account Management**: Add, edit, delete Kiro accounts
- **Balance Queries**: View remaining balance for each account
- **Token Refresh**: Manually refresh account tokens
- **Log Viewing**: View service operation logs
- **Config Management**: Edit service configuration online

---

## FAQ

**Q: Service starts but shows "Loaded 0 account configurations"**

You need to create `data/credentials.json` and add at least one account configuration. See [Getting Kiro Accounts](#getting-kiro-accounts) section.

**Q: Requests return `INVALID_MODEL_ID`**

> ⚠️ **[CRITICAL]** IPs in mainland China cannot directly access Claude models. You must configure `proxyUrl` in `data/config.json` (e.g., `"proxyUrl": "http://host.docker.internal:7890"`), or use a server outside China. This is the most common issue for users in China.

**Q: When using GPT-5.6 series models (sol/terra/luna), thinking mode, output effort, or max_tokens settings don't seem to work**

GPT-5.6 series Kiro backend schema does not support `additionalModelRequestFields` (covering thinking / output_config effort / max_tokens sub-fields), similar to Claude 4.5 generation (Sonnet 4.5 / Opus 4.5 / Haiku 4.5), the entire field is skipped. This is a known limitation, not a bug in this project.

**Q: Requests return 401 Unauthorized**

The API Key used by the client does not match the `apiKey` in `config.json`. Check and align them.

**Q: Token refresh fails / Request errors**

Try changing `tlsBackend` to `native-tls` in `config.json` and restart the service.

**Q: Error when importing account via admin panel: `Cannot read properties of undefined (reading 'digest')`**

This issue was fixed in v2.7.3: `crypto.subtle` encryption API only works in HTTPS or localhost environments. Public IP + HTTP access triggers this error. From v2.7.3, it automatically falls back to pure JS implementation, no need to configure HTTPS. If you still see this error, please upgrade to the latest version.

**Q: Enterprise IdC account requests return 502, logs show `profileArn is required for this request`**

Enterprise IdC accounts require `profileArn` when calling Q endpoints, but the IdC Token refresh interface doesn't return this field and needs manual input. The admin panel "Add Account / Edit Account" dialog provides a **Profile ARN** input field. Enter a value like `arn:aws:codewhisperer:<region>:<account-id>:profile/<profile-id>`. The `profileArn` can be obtained from Kiro IDE local cache or `ListAvailableProfiles`, and its region must match the account's `apiRegion`. Social accounts generally don't need this.

**Q: Can sub API Key consumption quota be measured by real Kiro credits (instead of estimated dollars)**

Yes. When creating/editing sub API Keys, the quota unit can be "Estimated Dollars" or "Real Credits" (limitUnit: usd/credits). When selecting credits, quota is measured by real credits_used in usage records (falls back to estimated_cost × k_ref for old records without credits_used field). Defaults to usd for backward compatibility.

**Q: Container cannot access host proxy**

Ensure your `docker-compose.yml` includes the following configuration:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Then use `http://host.docker.internal:port` as the proxy URL in `config.json`.

**Q: How to view container logs?**

```bash
docker compose logs -f kiro2cc-proxy
```

**Q: How to enter container for debugging?**

```bash
docker compose exec kiro2cc-proxy sh
```

---

## Notes

1. `credentials.json` contains sensitive tokens. Do not commit to version control or share with others
2. The service automatically refreshes expired tokens without manual intervention
3. In multi-account mode, refreshed tokens are automatically written back to the file
4. Users in mainland China must configure a proxy to access Claude models
5. For Docker deployment, config files are in the `./data` directory and automatically mounted into the container

---

## Project Structure

```
kiro-proxy/
├── src/                    # Rust source code
├── admin-ui/               # Admin panel frontend
├── user-ui/                # User panel frontend
├── data/                   # Docker config directory
│   ├── config.json         # Service configuration
│   └── credentials.json    # Account configuration
├── config.example.json     # Configuration example
├── docker-compose.yml      # Docker Compose configuration
├── Dockerfile              # Docker image build file
└── README.md               # This document
```

---

## License

MIT

## Acknowledgments

This project is based on [kiro.rs](https://github.com/hank9999/kiro.rs). Thanks to the original author for open-sourcing it.
