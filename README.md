# arti-builds

**Arti**（Tor 官方 Rust 实现）的非官方跨平台 CI 构建仓库。

- 源码：crates.io 官方发布包 `arti`（当前 `ARTI_VERSION = 2.7.0`，`--locked` 构建，不改动任何源码）
- 触发与发布（**commit 推送不触发任何构建**）：
  - **编译**：每 15 天自动（锚定 2026-10-15，首中 10-30）或手动 dispatch（`release=false` 仅构建出 artifacts）
  - **发版**：每 30 天自动（首中 11-14，当天自动构建 + 发布）或手动 dispatch 勾选 `release=true`，或推 `arti-v*` tag（tag 名即 release 名）
  - 定时机制：每日 UTC 18:17 探针 job 秒级退出，命中 15/30 天节拍才启动 11 目标构建；30 天命中日必落在 15 天节拍上
  - schedule/dispatch 的 release 名 = `arti-v<版本>-r<YYYYMMDD>`；同名 release 已存在时自动 `--clobber` 追加上传，重跑安全
- 免责：非 Tor Project 官方二进制；来自官方源码 + 官方 Cargo.lock，TLS 栈统一 `rustls`（ring provider）。用于生产前请自行评估。

## 构建矩阵（11 个目标）

| 平台 | target | 方式 | 备注 |
|---|---|---|---|
| Linux x86_64 | `x86_64-unknown-linux-gnu` | 原生 | 全功能 + `journald`（journal 日志直写 systemd） |
| Linux ARM64 | `aarch64-unknown-linux-gnu` | cross | |
| Linux ARMv7 | `armv7-unknown-linux-gnueabihf` | cross | 树莓派 32 位系 |
| Linux x86 32 位 | `i686-unknown-linux-gnu` | cross | |
| Linux RISC-V 64 | `riscv64gc-unknown-linux-gnu` | cross | |
| Linux PowerPC64 LE | `powerpc64le-unknown-linux-gnu` | cross | |
| Linux x86_64 musl | `x86_64-unknown-linux-musl` | cross | **全静态**，无 glibc 依赖，适合任意发行版/容器 |
| Linux ARM64 musl | `aarch64-unknown-linux-musl` | cross | **全静态** |
| Windows x64 | `x86_64-pc-windows-msvc` | 原生 | 静态 CRT（不依赖 VC Redist） |
| Windows x86 32 位 | `i686-pc-windows-msvc` | 原生 | 静态 CRT |
| Windows ARM64 | `aarch64-pc-windows-msvc` | 原生 | 静态 CRT |

每个目标产物：`arti-v<版本>-<target>.tar.gz`（Linux）/ `.zip`（Windows）+ `.sha256`，内含二进制 + Arti 官方 README + LICENSE。

## 编译进的功能（"所有可用功能"的精确展开）

对照 arti 2.7.0 的 65 个 feature flag，按平台精确组合：

- **默认全量**：SOCKS5 代理、DNS 代理（`dns-proxy`）、桥接客户端（`bridge-client`）、可插拔传输客户端（`pt-client`，obfs4/snowflake 等）、onion 服务**客户端**、vanguards（层防御）、压缩目录（`compression`）、内存硬化（`harden`）、流控拥塞控制（`flowctl-cc`）
- **`full` 增量**：onion 服务**服务端**（`onion-service-service`）、RPC 控制面（`rpc`，Arti 新控制协议）、HTTP CONNECT 代理（`http-connect`）、密钥管理器（`keymgr`）、受限发现（`restricted-discovery`）、系统日志（`syslog`，仅 Linux）、OpenTelemetry 追加器、Counter-Galois-Onion 抗量子密码、ctor 密钥库（`ctor-keystore`）
- **`experimental` 增量**：onion 服务客户端 CLI（`hsc`）、POW 抗 DoS 全量（`hs-pow-full`）、hsc/hss 扩展协商、onion 服务 CLI 扩展（`onion-service-cli-extra`）、实验 API（`experimental-api`）、Prometheus 指标导出（`metrics`）、tokio-console 调试、OpenTelemetry OTLP
- **链接选项**：`static-sqlite`（SQLite 静态编译，不依赖系统 libsqlite3）
- **TLS**：全部目标统一 `rustls-ring`（三选一互斥里交叉兼容性最好的组合）

### 有意排除的 feature（及原因）

| Feature | 原因 |
|---|---|
| `journald`（仅 Linux x86_64-gnu 有） | 依赖 libsystemd；Windows/musl/交叉环境无 systemd |
| `syslog`（仅 Windows 无） | syslog 传输依赖 unix socket |
| `async-std` | 与默认 `tokio` 冗余的第二套 runtime（`full` 会拉它），只增体积不增能力 |
| `rustls-aws-lc-rs` / `rustls` | aws-lc-sys 需要 cmake/go，且不支持 riscv64/ppc64le 等交叉目标 |
| `native-tls` / `static-native-tls` | Linux 上需要交叉编译 OpenSSL（大坑）；Windows 上 schannel 与 rustls 功能等价 |
| `static`（arti-client） | 语义不明且标记 non-additive；musl 本身已全静态，Windows 用 `+crt-static` |

### OpenSSL 处理说明

TLS 主栈是 rustls-ring，但 `opentelemetry`（OTLP 导出）→ `reqwest` → `native-tls` 的间接链会把 `openssl-sys` 拖进依赖树。为保全 OTLP 功能，**构建时向 Cargo.toml 注入一行 `openssl-sys = { features = ["vendored"] }`（仅 Linux 目标）**，OpenSSL 源码随二进制静态编译、不依赖系统 libssl；随后 `cargo update -p arti` 刷新锁文件以维持 `--locked` 构建。Windows 的 native-tls 走系统 schannel，不做注入（避免引入 nasm/openssl 构建负担）。

## 使用

```bash
tar xzf arti-v2.7.0-x86_64-unknown-linux-musl.tar.gz
cd arti-v2.7.0-x86_64-unknown-linux-musl
./arti proxy                 # SOCKS5 监听 127.0.0.1:9150
# 配置文件：~/.config/arti/arti.toml（root: /root/.config/arti/arti.toml）
# 例：改 SOCKS 端口 → [proxy] socks_listen = ["127.0.0.1:9050"]
# 例：RPC → [rpc] rpc_listen = ["127.0.0.1:8765"]
```

注意：实验性 feature（`experimental-*`、`hsc`、`metrics`、`tokio-console` 等）按 Tor Project 定义属于**不稳定 API 面**，行为可能随版本变化。

## 复现 / 升级

1. 改 `.github/workflows/build.yml` 顶部 `env.ARTI_VERSION`
2. Actions 页手动 dispatch，或直接 push

## License

Arti 源码与产物遵循上游 `MIT OR Apache-2.0`（随包附带 LICENSE 文件）。本仓库工作流代码无附加限制。
