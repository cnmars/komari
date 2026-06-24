# Komari 学习指南

## 项目概述

Komari 是一个轻量级、自托管的服务器监控工具，使用 Go 语言开发。它通过 Web 界面展示服务器状态，并通过轻量级 Agent 采集数据。

- **GitHub**: https://github.com/cnmars/komari
- **官方文档**: https://komari-document.pages.dev/
- **协议**: MIT License
- **技术栈**: Go 1.25 + Gin + GORM + SQLite + WebSocket + gRPC

---

## 目录

1. [项目架构](#1-项目架构)
2. [技术栈详解](#2-技术栈详解)
3. [源代码结构](#3-源代码结构)
4. [核心功能模块](#4-核心功能模块)
5. [学习路径](#5-学习路径)
6. [实战练习](#6-实战练习)
7. [参考资料](#7-参考资料)

---

## 1. 项目架构

```
┌─────────────────────────────────────────────────┐
│                    Web 浏览器                      │
│          (React/Vue 前端主题, Web界面)              │
└──────────────────┬──────────────────────────────┘
                   │ HTTP/WebSocket
                   ▼
┌─────────────────────────────────────────────────┐
│           Komari Server (Go + Gin)               │
│                                                   │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ 认证系统  │  │  RPC v2  │  │ 配置管理      │  │
│  │ (会话+2FA) │  │ (JSON-RPC)│  │ (KV存储)     │  │
│  └──────────┘  └──────────┘  └───────────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Agent管理 │  │ 通知系统  │  │ 任务调度      │  │
│  │          │  │(多通道)   │  │ (定时任务)    │  │
│  └──────────┘  └──────────┘  └───────────────┘  │
└──────────────────┬──────────────────────────────┘
                   │ gRPC / WebSocket
                   ▼
┌─────────────────────────────────────────────────┐
│              Komari Agent (客户端)                 │
│      (系统指标采集 + 远程命令执行 + 隧道)           │
└─────────────────────────────────────────────────┘
```

### 架构特点

- **B/S 架构**: 浏览器/服务器模式，通过 Web 界面管理
- **双协议支持**: Agent 可通过 gRPC (Nezha 兼容) 或 WebSocket (v2 协议) 上报数据
- **插件化通知**: 支持 Bark、邮件、ServerChan、JavaScript 脚本等多种通知方式
- **主题系统**: 前端完全可定制主题，支持用户自定义

---

## 2. 技术栈详解

### 2.1 Go 语言 (1.25)

**作用**: 整个后端服务使用 Go 编写

**学习要点**:
- **Goroutine + Channel**: 并发处理 Agent 连接和通知发送
- **`sync.Once`**: 单例模式 (数据库连接初始化)
- **`sync.RWMutex`**: 读写锁保护配置订阅者列表
- **`crypto/rand`**: 安全的随机数生成 (会话令牌、密码生成)
- **`crypto/sha256`**: 密码哈希
- **`net/http`**: HTTP 服务器
- **`reflect`**: 反射用于泛型配置的序列化/反序列化

**相关文件**:
- `cmd/server.go` - 服务启动入口
- `database/dbcore/dbcore.go` - 数据库连接 (单例模式)
- `pkg/config/config.go` - 泛型配置管理 (反射使用典范)

### 2.2 Gin Web 框架

**作用**: HTTP API 路由和中间件

**学习要点**:
- **路由分组**: `/api/public`, `/api/admin` 等
- **中间件链**: 认证 → 日志 → 2FA 验证
- **请求绑定**: `c.ShouldBindJSON()`, `c.Query()`
- **上下文传递**: `c.Set()` / `c.Get()` 在中间件间传递认证信息
- **Cookie 操作**: `c.Cookie()`, `c.SetCookie()`

**相关文件**:
- `web/api/Auth.go` - 认证中间件
- `web/api/public/login.go` - 登录路由
- `utils/gin.go` - Gin 工具函数
- `utils/log/gin.go` - Gin 日志中间件

```go
// 中间件链示例 (web/api/Auth.go)
r.Use(middleware.AuthRequired())
// 路由分组
adminGroup := r.Group("/api/admin")
adminGroup.Use(middleware.RequireSensitive2FA())
```

### 2.3 GORM (ORM)

**作用**: 数据库操作 (主要使用 SQLite)

**学习要点**:
- **自动迁移**: `db.AutoMigrate(&models.User{}, ...)`
- **模型定义**: gorm tag `gorm:"primaryKey;column:key;type:text"`
- **链式查询**: `db.Where("key IN ?", keys).Find(&items)`
- **Upsert**: `clause.OnConflict{DoUpdates: clause.AssignmentColumns([]string{"value"})}`
- **原始 SQL**: `db.Exec("PRAGMA journal_mode = WAL;")`
- **事务**: 隐式事务 (单条操作)

**相关文件**:
- `database/dbcore/dbcore.go` - 数据库初始化
- `database/models/models.go` - 数据模型
- `pkg/config/config.go` - 配置持久化
- `database/accounts/accounts.go` - 账户 CRUD

### 2.4 WebSocket (gorilla/websocket)

**作用**: Agent 实时通信和终端代理

**学习要点**:
- **Upgrader**: HTTP → WebSocket 升级
- **Read/Write 循环**: 并发读写 (goroutine)
- **Origin 校验**: 安全检查
- **心跳检测**: Ping/Pong 维持连接

**相关文件**:
- `web/api/WebSocket.go` - WebSocket 处理
- `protocol/v2/jsonrpc.go` - JSON-RPC over WebSocket
- `protocol/v1/report.go` - v1 协议上报

### 2.5 goja (JavaScript 引擎)

**作用**: 执行用户自定义通知脚本

**学习要点**:
- **沙箱执行**: 纯 Go 实现的 JS 引擎，不支持 `require`
- **API 注入**: 通过 `Set` 方法注入自定义函数
- **超时控制**: `context.WithTimeout` 防止无限循环
- **内存限制**: 通过 `SetMaxStack` 限制执行栈

**相关文件**:
- `utils/messageSender/javascript/javascript.go` - JS 执行引擎
- `utils/messageSender/javascript/apis.go` - JS API 定义

```go
// JS 引擎注入示例
vm.Set("fetch", func(call goja.FunctionCall) goja.Value {
    // 自定义 fetch 实现
})
vm.Set("console", consoleObj)
```

### 2.6 gRPC (Google gRPC)

**作用**: Nezha 兼容层 (Agent 通信)

**学习要点**:
- **Protocol Buffers**: 消息序列化
- **双向流**: gRPC Stream 用于实时数据上报
- **TLS/mTLS**: 可选的传输加密

**相关文件**:
- `web/nezha/` - Nezha 兼容层目录
- `protocol/` - 协议定义

### 2.7 Cobra (CLI 框架)

**作用**: 命令行参数解析

**学习要点**:
- **命令注册**: `RootCmd.AddCommand(serverCmd)`
- **标志绑定**: `RootCmd.PersistentFlags().StringVarP()`
- **环境变量默认值**: 通过 `os.Getenv()` 读取

**相关文件**:
- `cmd/root.go` - 根命令定义
- `cmd/server.go` - server 子命令

### 2.8 认证与安全

**主要机制**:
- **会话认证**: 基于 Cookie 的 session token (32 字符随机字符串)
- **API Key**: 用于自动化管理的 Bearer Token
- **2FA**: TOTP 双因素认证
- **OAuth SSO**: 支持 QQ、Google、GitHub 等 OAuth 登录

**相关文件**:
- `database/accounts/sessions.go` - 会话管理
- `database/accounts/2fa.go` - 双因素认证
- `web/api/Auth.go` - 认证中间件
- `web/api/public/oauth.go` - OAuth 登录

---

## 3. 源代码结构

```
komari/
├── main.go                    # 程序入口
├── go.mod / go.sum            # Go 依赖管理
├── Dockerfile                 # Docker 构建
├── install-komari.sh          # 一键安装脚本
│
├── cmd/                       # 命令行
│   ├── root.go                # 根命令 (参数定义)
│   ├── server.go              # server 子命令 (启动服务)
│   ├── chpasswd.go            # 修改密码
│   ├── disable2FA.go          # 禁用 2FA
│   ├── permitPasswordLogin.go # 允许密码登录
│   └── flags/
│       └── config.go          # 配置标志定义
│
├── database/                  # 数据库层
│   ├── dbcore/
│   │   └── dbcore.go          # 数据库连接管理 (单例)
│   ├── models/
│   │   ├── models.go          # 核心数据模型
│   │   ├── clipboard.go       # 剪贴板模型
│   │   ├── log.go             # 日志模型
│   │   ├── notification.go    # 通知模型
│   │   ├── oauth.go           # OAuth 模型
│   │   ├── pingTask.go        # Ping 任务模型
│   │   ├── task.go            # 任务模型
│   │   ├── theme.go           # 主题模型
│   │   └── UTCTime.go         # 自定义时间类型
│   ├── accounts/
│   │   ├── accounts.go        # 账户 CRUD
│   │   ├── sessions.go        # 会话管理
│   │   └── 2fa.go             # 2FA 认证
│   ├── clients/
│   │   ├── client.go          # Agent 客户端管理
│   │   └── report.go          # 客户端数据上报
│   ├── records/
│   │   └── records.go         # 监控记录
│   ├── tasks/
│   │   ├── tasks.go           # 任务管理
│   │   └── ping.go            # Ping 任务
│   ├── notification/
│   │   ├── load.go            # 负载通知
│   │   └── traffic_report.go  # 流量报告
│   ├── clipboard/
│   │   └── clipboard.go       # 剪贴板功能
│   ├── auditlog/
│   │   └── log.go             # 审计日志
│   ├── oauth.go               # OAuth 配置
│   ├── messageSender.go       # 消息发送
│   └── utils.go               # 数据库工具
│
├── pkg/                       # 核心包
│   ├── config/
│   │   ├── config.go          # 配置管理 (泛型 KV 存储)
│   │   └── settings.go        # 设置定义
│   ├── corn/
│   │   └── corn.go            # 定时任务调度
│   ├── metric/
│   │   ├── store.go           # 指标存储
│   │   ├── types.go           # 指标类型
│   │   ├── rollup.go          # 数据聚合
│   │   ├── rollup_store.go    # 聚合存储
│   │   ├── rollup_query.go    # 聚合查询
│   │   ├── tdigest.go         # T-Digest 百分位计算
│   │   ├── dialect.go         # SQL 方言
│   │   ├── aggregate.go       # 聚合函数
│   │   ├── json.go            # JSON 工具
│   │   ├── migrations.go      # 指标迁移
│   │   └── config.go          # 指标配置
│   ├── migrations/
│   │   ├── migrations.go      # 数据库迁移
│   │   └── legacy.go          # 旧版迁移兼容
│   └── rpc/
│       ├── rpc.go             # RPC 核心
│       ├── context.go         # RPC 上下文
│       ├── request.go         # RPC 请求处理
│       ├── response.go        # RPC 响应
│       ├── registry.go        # RPC 方法注册
│       ├── invoke.go          # RPC 调用
│       ├── permission.go      # RPC 权限控制
│       ├── parse.go           # JSON-RPC 解析
│       ├── meta.go            # RPC 元数据
│       ├── internal.go        # 内部 RPC
│       └── errors.go          # RPC 错误定义
│
├── protocol/                  # 通信协议
│   ├── v1/
│   │   └── report.go          # v1 报告协议
│   └── v2/
│       ├── jsonrpc.go         # v2 JSON-RPC 协议
│       └── networktest.go     # 网络测试协议
│
├── utils/                     # 工具包
│   ├── cloudflared/
│   │   └── cloudflared.go     # Cloudflare Tunnel
│   ├── geoip/
│   │   ├── geoip.go           # IP 地理位置
│   │   ├── mmdb.go            # MaxMind DB 解析
│   │   ├── geojs.js           # 前端 GeoJS
│   │   ├── ipapi.go           # ipapi.co 查询
│   │   ├── ipinfo.go          # ipinfo.io 查询
│   │   └── emptyProvider.go   # 空提供者
│   ├── item/
│   │   └── item.go            # 数据项工具
│   ├── log/
│   │   ├── log.go             # 日志配置
│   │   ├── gin.go             # Gin 日志中间件
│   │   └── gorm.go            # GORM 日志
│   ├── messageSender/
│   │   ├── sender.go          # 发送器核心
│   │   ├── loader.go          # 加载器
│   │   ├── all.go             # 全部注册
│   │   ├── factory/
│   │   │   ├── factory.go     # 工厂模式
│   │   │   └── meta.go        # 元数据
│   │   ├── bark/
│   │   │   ├── bark.go        # Bark 推送
│   │   │   └── meta.go
│   │   ├── email/
│   │   │   ├── email.go       # 邮件发送
│   │   │   └── meta.go
│   │   ├── empty/
│   │   │   └── empty.go       # 空发送器
│   │   ├── javascript/
│   │   │   ├── javascript.go  # JS 引擎执行
│   │   │   ├── apis.go        # JS API 注入
│   │   │   └── meta.go
│   │   ├── serverchan3/
│   │   │   ├── serverchan3.go # ServerChan v3
│   │   │   └── meta.go
│   │   └── serverchanturbo/
│   │       ├── serverchanturbo.go # ServerChan Turbo
│   │       └── meta.go
│   └── gin.go                 # Gin 工具
│
└── web/                       # Web 接口 (由 komari-web 子模块提供)
    └── public/
        └── defaultTheme/      # 默认主题 (前端静态文件)
```

---

## 4. 核心功能模块

### 4.1 认证系统

```
┌─────────┐     ┌──────────────┐     ┌───────────┐
│ 用户输入  │────▶│ Auth 中间件   │────▶│ Session   │
│ 用户名   │     │ (web/api/    │     │ 管理      │
│ 密码     │     │  Auth.go)    │     │ (sessions │
└─────────┘     └──────────────┘     │ .go)      │
                                       └───────────┘
```

**认证流程**:
1. 用户 POST `/api/login` 提交用户名和密码
2. 服务端用 SHA256(salt+password) 验证密码
3. 成功后创建 Session (32 字符随机 Token)
4. Session Token 存入 Cookie (`session_token`)
5. 后续请求通过 Auth 中间件验证 Cookie

**支持方式**:
- 密码登录
- OAuth SSO (QQ/Google/GitHub)
- API Key (Bearer Token)
- 2FA TOTP (Google Authenticator)

### 4.2 Agent 通信协议

**v1 协议**: 简单 HTTP POST 上报
- Agent 通过 HTTP POST 向 `/api/v1/report` 发送 JSON 数据
- 使用 UUID + Secret 进行身份验证

**v2 协议**: JSON-RPC over WebSocket
- 基于 WebSocket 的 JSON-RPC 2.0
- 支持双向通信 (Server → Agent 指令)
- 支持终端代理和远程控制

**Nezha 兼容层**: gRPC
- 兼容原 Nezha 监控 Agent
- 支持无缝迁移

### 4.3 指标系统

```
┌──────────┐    ┌───────────┐    ┌──────────────┐
│ Agent    │───▶│ 原始记录   │───▶│ 数据聚合     │
│ 上报数据  │    │ (records) │    │ (rollup)     │
└──────────┘    └───────────┘    └──────────────┘
                                         │
                                         ▼
                                  ┌──────────────┐
                                  │ 长期存储      │
                                  │ (long_term)   │
                                  └──────────────┘
```

**采集指标**:
- CPU 使用率
- 内存使用率
- 磁盘 IO
- 网络流量
- GPU 指标
- 进程信息
- 负载均衡

**数据聚合**: 使用 T-Digest 算法计算百分位数，支持自动数据 Rollup (短期高精度 → 长期低精度)

### 4.4 通知系统

**工厂模式设计**:

```
┌────────────────────────┐
│  MessageSender Factory  │
│  (utils/messageSender/  │
│   factory/)             │
└──────────┬─────────────┘
           │
    ┌──────┼──────┐
    ▼      ▼      ▼
┌──────┐ ┌──────┐ ┌──────────┐
│ Bark │ │ 邮件  │ │ Server  │
│      │ │      │ │ Chan    │
└──────┘ └──────┘ └──────────┘
    ▼      ▼      ▼
┌──────────────────────────┐
│  JavaScript 脚本引擎      │
│  (goja 沙箱执行)          │
└──────────────────────────┘
```

**触发条件**:
- 服务器离线/恢复
- CPU/内存/磁盘负载过高
- 流量报告 (日/周/月)
- 安全事件 (登录通知)

### 4.5 任务系统

- **定时 Ping**: 监控目标主机延迟和可用性
- **自定义脚本**: 通过 Agent 执行远程命令
- **Cron 调度**: 内置的定时任务调度器

### 4.6 API 分层

```
/api/public/*    - 公开接口 (登录、OAuth)
/api/admin/*     - 管理接口 (需管理员权限)
/api/rpc2        - JSON-RPC v2 (需认证)
/api/ws          - WebSocket 终端/隧道
/api/v1/*        - Nezha v1 兼容接口
```

---

## 5. 学习路径

### 阶段 1: Go 语言基础 (1-2 周)

| 知识点 | 说明 | 练习项目 |
|--------|------|----------|
| 基础语法 | 变量、控制流、函数 | LeetCode Easy |
| 并发编程 | goroutine, channel, select | 简易并发下载器 |
| 标准库 | net/http, encoding/json, crypto | HTTP 客户端 |
| 接口与泛型 | interface, 类型参数 | 通用缓存库 |

### 阶段 2: Web 开发基础 (2-3 周)

| 知识点 | 对应 Komari 代码 | 练习项目 |
|--------|-----------------|----------|
| Gin 路由 | `web/api/*.go` | RESTful API |
| 中间件 | `web/api/Auth.go` | 日志/认证中间件 |
| GORM | `database/*.go` | CRUD API |
| WebSocket | `web/api/WebSocket.go` | 聊天室 |
| Cookie/Session | `database/accounts/sessions.go` | 用户认证系统 |

### 阶段 3: Komari 核心模块 (3-4 周)

1. **认证模块** (`database/accounts/`)
   - 研究 `CheckPassword` → `CreateSession` → Auth 中间件流程
   - 理解 2FA TOTP 实现 (`pquerna/otp`)
   - 掌握 OAuth 2.0 授权码流程

2. **RPC 系统** (`pkg/rpc/`)
   - 理解 JSON-RPC 2.0 协议规范
   - 方法注册机制 (`Register`) → 权限控制 (`CheckPermission`)
   - 前端通过 RPC v2 获取所有数据

3. **指标存储** (`pkg/metric/`)
   - 理解 T-Digest 百分位算法
   - 数据 rollup 策略 (10s → 10min → 1h)
   - SQL 方言适配器模式

4. **通知系统** (`utils/messageSender/`)
   - 工厂模式 + 插件架构
   - JavaScript 沙箱运行通知脚本
   - 各通道适配器实现

### 阶段 4: 前端开发 (2-3 周)

Komari 前端是一个独立的 Vue 3 项目:
```bash
git clone https://github.com/komari-monitor/komari-web
cd komari-web
npm install
npm run build
```

- **主题系统**: 通过 `komari-theme.json` 自定义颜色和布局
- **RPC 通信**: 前端通过 JSON-RPC v2 与后端交互
- **WebSocket**: 实时数据更新

### 阶段 5: 安全实践 (1-2 周)

- 阅读本项目安全分析文档 `SECURITY_AUDIT.md`
- 修复建议中的漏洞作为实战练习
- 学习 OWASP Top 10 在 Go 中的表现

---

## 6. 实战练习

### 练习 1: 搭建开发环境

```bash
# 后端
git clone https://github.com/cnmars/komari
cd komari
go build -o komari
./komari server -l 0.0.0.0:25774

# 前端 (需要 Node.js 20+)
git clone https://github.com/komari-monitor/komari-web
cd komari-web
npm install
npm run build
# 将 dist 复制到 komari/web/public/defaultTheme/
```

### 练习 2: 添加一个新的通知通道

1. 在 `utils/messageSender/` 下创建新目录
2. 实现 `Sender` 接口
3. 在 `all.go` 中注册
4. 添加工厂元数据

### 练习 3: 实现 API 限流中间件

1. 基于 Gin 中间件
2. 使用令牌桶算法或滑动窗口
3. 按 IP 和用户双重限制

### 练习 4: 安全漏洞修复

参考安全分析报告中的 C-01 (弱密码哈希):
1. 用 `bcrypt` 替换 `SHA256+固定盐`
2. 确保向后兼容 (旧密码验证逻辑)
3. 添加密码强度验证

---

## 7. 参考资料

### Go
- [官方教程](https://go.dev/doc/tutorial/)
- [Go 标准库文档](https://pkg.go.dev/std)

### 框架和库
- [Gin Web Framework](https://gin-gonic.com/)
- [GORM](https://gorm.io/)
- [gorilla/websocket](https://github.com/gorilla/websocket)
- [goja (JS Engine)](https://github.com/dop251/goja)
- [cobra](https://github.com/spf13/cobra)
- [gRPC Go](https://grpc.io/docs/languages/go/)

### 协议
- [JSON-RPC 2.0](https://www.jsonrpc.org/specification)
- [TOTP (RFC 6238)](https://datatracker.ietf.org/doc/html/rfc6238)
- [OAuth 2.0](https://oauth.net/2/)

### 安全
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Go 安全最佳实践](https://go.dev/security/)

### 相关项目
- [Nezha (哪吒监控)](https://github.com/naiba/nezha) - Komari 的前身
- [Huntress Komari 分析](https://www.huntress.com/blog/komari-c2-agent-abuse)

---

*本指南由 AI 自动生成，基于 komari 项目源代码分析*
