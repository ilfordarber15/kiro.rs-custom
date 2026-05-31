# kiro.rs-custom

一个用 Rust 编写的 Anthropic Claude API 兼容代理服务，将 Anthropic API 请求转换为 Kiro API 请求。

> **本项目是 [hank9999/kiro.rs](https://github.com/hank9999/kiro.rs) 的 Fork**，目的是基于上游最新 master 编译，确保支持新发布的模型（如 Claude Opus 4.7 / 4.8）。所有核心代码版权归原作者 hank9999 所有。

---

## 与上游的差异

本仓库与上游唯一的实质差异：

- 移除了 README 中的赞助商广告位，使展示更简洁
- 在 `Cargo.toml` 的 `[profile.release]` 中关闭了 LTO（`lto = false`），方便在低内存（< 1GB）的机器上完成 release 编译。如对二进制体积/性能敏感，可改回 `lto = true` 后在大内存机器上编译

源码逻辑、协议转换、模型映射等与上游保持一致。

## 免责声明

本项目仅供研究使用，Use at your own risk，使用本项目所导致的任何后果由使用人承担，与本项目及上游作者无关。
本项目与 AWS / KIRO / Anthropic / Claude 等官方无关，本项目不代表官方立场。

## 注意

- TLS 默认使用 `rustls`。如遇 token 刷新失败或代理问题，可在 `config.json` 中将 `tlsBackend` 切换为 `native-tls` 后重试
- 若遇到 Write Failed / 会话卡死，参考上游 Issue [#22](https://github.com/hank9999/kiro.rs/issues/22) 与 [#49](https://github.com/hank9999/kiro.rs/issues/49)

## 功能特性

- **Anthropic API 兼容**：完整支持 Anthropic Claude API 格式
- **流式响应**：支持 SSE（Server-Sent Events）流式输出
- **Token 自动刷新**：自动管理和刷新 OAuth Token
- **多凭据支持**：支持配置多个凭据，按优先级自动故障转移
- **负载均衡**：支持 `priority`（按优先级）和 `balanced`（均衡分配）两种模式
- **智能重试**：单凭据最多重试 3 次，单请求最多重试 9 次
- **凭据回写**：多凭据格式下自动回写刷新后的 Token
- **Thinking 模式**：支持 Claude 的 extended thinking 功能
- **工具调用**：完整支持 function calling / tool use
- **WebSearch**：内置 WebSearch 工具转换逻辑
- **多模型支持**：覆盖 Sonnet 4.5/4.6、Opus 4.5/4.6/4.7/4.8、Haiku 4.5（含 -thinking 变体）
- **Admin 管理**：可选的 Web 管理界面和 API，支持凭据管理、余额查询等
- **多级 Region 配置**：支持全局和凭据级别的 Auth Region / API Region 配置
- **凭据级代理**：支持为每个凭据单独配置 HTTP/SOCKS5 代理

---

- [开始](#开始)
  - [1. 编译](#1-编译)
  - [2. 最小配置](#2-最小配置)
  - [3. 启动](#3-启动)
  - [4. 验证](#4-验证)
  - [Docker](#docker)
- [配置详解](#配置详解)
- [API 端点](#api-端点)
- [模型映射](#模型映射)
- [Admin（可选）](#admin可选)
- [项目结构](#项目结构)
- [技术栈](#技术栈)
- [License](#license)
- [致谢](#致谢)

## 开始

### 1. 编译

> **前置步骤**：编译前需先构建前端 Admin UI（其产物会通过 `rust-embed` 嵌入二进制中，缺失会导致编译失败）：
> ```bash
> cd admin-ui && npm install && npm run build
> # 或：pnpm install && pnpm build
> cd ..
> ```

```bash
cargo build --release
```

**Rust 版本要求**：>= 1.85（项目使用 `edition = "2024"`，旧版 Cargo 会报 `edition2024 is required`）。如目标机器版本过低，可执行 `rustup update stable` 升级。

**低内存机器编译说明**：本仓库已默认关闭 LTO，约 1GB 内存即可编译完成。若需要追求体积/性能，可将 `Cargo.toml` 中 `lto = false` 改回 `true`。

编译产物位于：`target/release/kiro-rs`

### 2. 最小配置

创建 `config.json`：

```json
{
   "host": "127.0.0.1",
   "port": 8990,
   "apiKey": "sk-your-api-key",
   "region": "us-east-1"
}
```

> 如需 Web 管理面板，请配置 `adminApiKey`。

创建 `credentials.json`（从 Kiro IDE 等中获取凭证信息）：

Social 认证：
```json
{
   "refreshToken": "你的刷新token",
   "expiresAt": "2027-01-01T00:00:00.000Z",
   "authMethod": "social"
}
```

IdC 认证：
```json
{
   "refreshToken": "你的刷新token",
   "expiresAt": "2027-01-01T00:00:00.000Z",
   "authMethod": "idc",
   "clientId": "你的clientId",
   "clientSecret": "你的clientSecret"
}
```

### 3. 启动

```bash
./target/release/kiro-rs
```

或指定配置文件路径：

```bash
./target/release/kiro-rs -c /path/to/config.json --credentials /path/to/credentials.json
```

### 4. 验证

```bash
# 列出可用模型
curl http://127.0.0.1:8990/v1/models -H "x-api-key: sk-your-api-key"

# 发送一条消息
curl http://127.0.0.1:8990/v1/messages \
  -H "Content-Type: application/json" \
  -H "x-api-key: sk-your-api-key" \
  -d '{
    "model": "claude-opus-4-8",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "Hello, Claude!"}
    ]
  }'
```

### Docker

仓库提供了 `Dockerfile` 和 `docker-compose.yml`。常见两种用法：

**用法 A：仓库内一键 build**

```bash
# 在仓库根目录
docker compose up -d --build
```

挂载 `config/` 目录后，将 `config.json` 和 `credentials.json` 放入即可。

**用法 B：先编译，再用轻量基础镜像打包**

适合在性能机上编译、再把二进制部署到生产机的场景。最小 Dockerfile 示例：

```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY kiro-rs ./
RUN chmod +x ./kiro-rs
CMD ["./kiro-rs", "-c", "/app/config/config.json"]
```

## 配置详解

### config.json

| 字段 | 类型 | 默认值 | 描述 |
|------|------|--------|------|
| `host` | string | `127.0.0.1` | 服务监听地址 |
| `port` | number | `8080` | 服务监听端口 |
| `apiKey` | string | - | 客户端访问本服务时使用的 API Key（必配） |
| `region` | string | `us-east-1` | AWS 区域 |
| `authRegion` | string | - | Auth Region（用于 Token 刷新），未配置时回退到 `region` |
| `apiRegion` | string | - | API Region（用于 API 请求），未配置时回退到 `region` |
| `kiroVersion` | string | `0.9.2` | Kiro 版本号 |
| `machineId` | string | - | 自定义机器码（64位十六进制），未指定则自动生成 |
| `systemVersion` | string | 随机 | 系统版本标识 |
| `nodeVersion` | string | `22.21.1` | Node.js 版本标识 |
| `tlsBackend` | string | `rustls` | TLS 后端：`rustls` 或 `native-tls` |
| `countTokensApiUrl` | string | - | 外部 count_tokens API 地址 |
| `countTokensApiKey` | string | - | 外部 count_tokens API 密钥 |
| `countTokensAuthType` | string | `x-api-key` | 外部 API 认证类型：`x-api-key` 或 `bearer` |
| `proxyUrl` | string | - | HTTP/SOCKS5 代理地址 |
| `proxyUsername` | string | - | 代理用户名 |
| `proxyPassword` | string | - | 代理密码 |
| `adminApiKey` | string | - | Admin API 密钥，配置后启用凭据管理 API 与 Web 管理界面 |
| `loadBalancingMode` | string | `priority` | 负载均衡模式：`priority` 或 `balanced` |
| `extractThinking` | boolean | `true` | 非流式响应中将 `<thinking>` 解析为独立 thinking 块 |
| `defaultEndpoint` | string | `ide` | 默认 Kiro 端点。当前仅支持 `ide` |

### credentials.json

支持单对象格式（向后兼容）或数组格式（多凭据）。详细字段请参考仓库内 `credentials.example.*.json` 文件。

多凭据特性：

- 按 `priority` 字段排序，数字越小优先级越高
- 单凭据最多重试 3 次，单请求最多重试 9 次
- 自动故障转移到下一个可用凭据
- 多凭据格式下 Token 刷新后自动回写到源文件

### Region 配置

**Auth Region**（Token 刷新）优先级：
`凭据.authRegion` > `凭据.region` > `config.authRegion` > `config.region`

**API Region**（API 请求）优先级：
`凭据.apiRegion` > `config.apiRegion` > `config.region`

### 代理配置

支持全局代理和凭据级代理，凭据级代理会覆盖该凭据产生的所有出站连接。

**优先级**：`凭据.proxyUrl` > `config.proxyUrl` > 无代理

| 凭据 `proxyUrl` 值 | 行为 |
|---|---|
| 具体 URL（`http://proxy:8080`、`socks5://proxy:1080`） | 使用凭据指定的代理 |
| `direct` | 显式不使用代理（即使全局已配置） |
| 未配置（留空） | 回退到全局代理配置 |

### 客户端认证方式

向本服务发送请求时，支持两种认证方式：

```
x-api-key: sk-your-api-key
```

或：

```
Authorization: Bearer sk-your-api-key
```

### 环境变量

```bash
RUST_LOG=debug ./target/release/kiro-rs
```

## API 端点

### 标准端点 (/v1)

| 端点 | 方法 | 描述 |
|------|------|------|
| `/v1/models` | GET | 获取可用模型列表 |
| `/v1/messages` | POST | 创建消息 |
| `/v1/messages/count_tokens` | POST | 估算 Token 数量 |

### Claude Code 兼容端点 (/cc/v1)

| 端点 | 方法 | 描述 |
|------|------|------|
| `/cc/v1/messages` | POST | 创建消息（缓冲模式，确保 `input_tokens` 准确） |
| `/cc/v1/messages/count_tokens` | POST | 估算 Token 数量 |

> `/cc/v1/messages` 与 `/v1/messages` 的区别：前者会缓冲整个上游响应，用 `contextUsageEvent` 计算的精确 `input_tokens` 更正 `message_start` 后再下发，期间每 25 秒发送 `ping` 保活。

### Thinking 模式

```json
{
  "model": "claude-opus-4-8",
  "max_tokens": 16000,
  "thinking": {
    "type": "enabled",
    "budget_tokens": 10000
  },
  "messages": [...]
}
```

或使用 `-thinking` 后缀的模型 ID（如 `claude-opus-4-8-thinking`）。

### 工具调用

完整支持 Anthropic 的 tool use 协议，请求格式与上游一致。

## 模型映射

模型映射通过子串匹配实现。客户端传入的模型名只要包含对应关键字与版本号，即会被映射到对应的 Kiro 模型：

| 客户端模型名（包含的关键字） | 映射后的 Kiro 模型 |
|---|---|
| `sonnet` + `4-6` 或 `4.6` | `claude-sonnet-4.6` |
| `sonnet` + `4-5` 或 `4.5` | `claude-sonnet-4.5` |
| `opus` + `4-5` 或 `4.5` | `claude-opus-4.5` |
| `opus` + `4-6` 或 `4.6` | `claude-opus-4.6` |
| `opus` + `4-7` 或 `4.7` | `claude-opus-4.7` |
| `opus` + `4-8` 或 `4.8` | `claude-opus-4.8` |
| `haiku` | `claude-haiku-4.5` |

`-thinking` 后缀不影响匹配。`/v1/models` 返回的标准模型 ID 列表见接口实际返回值。

**上下文窗口**：4.6 / 4.7 / 4.8 系列均为 1M tokens，其余为 200K tokens。

## Admin（可选）

当 `config.json` 配置了非空 `adminApiKey` 时启用。

**Admin API**（认证同主 API）：

- `GET /api/admin/credentials` 获取所有凭据状态
- `POST /api/admin/credentials` 添加新凭据
- `DELETE /api/admin/credentials/:id` 删除凭据
- `POST /api/admin/credentials/:id/disabled` 设置凭据禁用状态
- `POST /api/admin/credentials/:id/priority` 设置凭据优先级
- `POST /api/admin/credentials/:id/reset` 重置失败计数
- `GET /api/admin/credentials/:id/balance` 获取凭据余额

**Admin UI**：

- `GET /admin` Web 管理界面（依赖编译前已构建好的 `admin-ui/dist`）

## 项目结构

```
.
├── src/
│   ├── main.rs                  # 程序入口
│   ├── http_client.rs           # HTTP 客户端构建
│   ├── token.rs                 # Token 计算
│   ├── debug.rs / test.rs       # 调试与测试
│   ├── model/                   # 配置和参数模型
│   ├── anthropic/               # Anthropic API 兼容层（路由、协议转换、流处理、WebSearch）
│   ├── kiro/                    # Kiro API 客户端（provider、token 管理、AWS event-stream 解析器）
│   ├── admin/                   # Admin API（路由、handlers、service、auth）
│   ├── admin_ui/                # 前端静态文件嵌入
│   └── common/                  # 公共模块
├── admin-ui/                    # Admin UI 前端工程（构建产物嵌入二进制）
├── tools/                       # 辅助工具
├── Cargo.toml / Cargo.lock
├── config.example.json
├── credentials.example.*.json   # 不同认证方式的示例
├── docker-compose.yml
└── Dockerfile
```

## 技术栈

- **Web 框架**：[Axum](https://github.com/tokio-rs/axum) 0.8
- **异步运行时**：[Tokio](https://tokio.rs/)
- **HTTP 客户端**：[Reqwest](https://github.com/seanmonstar/reqwest)
- **序列化**：[Serde](https://serde.rs/)
- **日志**：[tracing](https://github.com/tokio-rs/tracing)
- **命令行**：[Clap](https://github.com/clap-rs/clap)
- **静态资源嵌入**：[rust-embed](https://github.com/pyrossh/rust-embed)

## License

MIT，与上游保持一致。详见 [LICENSE](./LICENSE)。

## 致谢

本项目是 [hank9999/kiro.rs](https://github.com/hank9999/kiro.rs) 的 Fork，所有核心实现归功于原作者及上游贡献者。

上游同样致谢的前置工作：

- [kiro2api](https://github.com/caidaoli/kiro2api)
- [proxycast](https://github.com/aiclientproxy/proxycast)
