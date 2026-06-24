# Komari 安全审计报告

**项目**: https://github.com/cnmars/komari (fork of komari-monitor/komari)
**审计日期**: 2026-06-24
**风险等级**: **高 (7.2/10)**

---

## 目录

1. [严重漏洞 (CVSS 9+)](#1-严重漏洞-cvss-9)
2. [高危漏洞 (CVSS 7-8.9)](#2-高危漏洞-cvss-7-89)
3. [中危漏洞 (CVSS 4-6.9)](#3-中危漏洞-cvss-4-69)
4. [低危漏洞 (CVSS 1-3.9)](#4-低危漏洞-cvss-1-39)
5. [与 Nezha 漏洞对比](#5-与-nezha-漏洞对比)
6. [安全加固建议](#6-安全加固建议)

---

## 1. 严重漏洞 (CVSS 9+)

### C-01: 弱密码哈希 - 硬编码固定盐 (CVSS 9.3)

**位置**: `database/accounts/accounts.go:13,80-86`

```go
const constantSalt = "06Wm4Jv1Hkxx"

func hashPasswd(passwd string) string {
    saltedPassword := passwd + constantSalt
    hash := sha256.New()
    hash.Write([]byte(saltedPassword))
    hashedPassword := base64.StdEncoding.EncodeToString(hash.Sum(nil))
    return hashedPassword
}
```

**问题**:
- 使用 SHA256 (无密钥拉伸，如 bcrypt/scrypt/argon2)
- 硬编码的全局固定盐 (`"06Wm4Jv1Hkxx"`)
- 无每用户随机盐
- GPU/ASIC 可在每秒数十亿次哈希攻击

**影响**: 数据库泄露后，密码可被快速破解。由于盐是固定的，所有用户的密码可并行破解。

**修复**: 使用 `golang.org/x/crypto/bcrypt` (cost >= 12) 或 argon2id 替换。为每个用户生成随机盐。

### C-02: Nezha gRPC 兼容层认证绕过 (CVSS 9.8)

**位置**: `cmd/server.go:74-78`, `web/nezha/`

**问题**: Nezha 兼容层引入了与原 Nezha 项目相同的 gRPC 通信协议，而 Nezha 已知存在:
- gRPC 认证绕过 (伪造客户端身份)
- Agent 注册无需所有权证明
- 通过构造的 gRPC 消息执行任意命令

**影响**: 攻击者可利用 Nezha 已知漏洞绕过认证，控制 Agent 或服务端。

**修复**: 为兼容层添加严格认证，使用 mTLS，并尽快弃用兼容层。

### C-03: JavaScript 通知引擎 SSRF (CVSS 9.0)

**位置**: `utils/messageSender/javascript/javascript.go:69-77`, `apis.go:30-130`

```go
client := &http.Client{Timeout: 30 * time.Second}
resp, err := client.Do(req)  // 可访问任意主机和端口
```

**问题**: JavaScript 沙箱暴露了 `fetch()` 和 `XMLHttpRequest` API，且没有任何目标地址限制。管理员配置的 JS 通知脚本可访问任意内部网络资源。

**影响**: SSRF 攻击，可访问:
- `http://localhost:25774/api/admin/*` (自身 API)
- `http://169.254.169.254/latest/meta-data/` (云元数据)
- 内部网络的其他服务

**修复**:
- 实现出站 HTTP 白名单
- 默认禁止私有 IP 地址范围
- 添加配置项限制允许的目标地址

### C-04: API Key 明文存储 (CVSS 9.0)

**位置**: `pkg/config/settings.go:28`, `pkg/config/config.go:21-23`

```go
type ConfigItem struct {
    Value string `gorm:"column:value;type:text"` // 明文 JSON 字符串
}
```

**问题**: API Key 以 JSON 字符串明文存储在 SQLite 数据库中。API Key 拥有完整的管理员权限 (`RoleAdmin`)，且可绕过 2FA 验证。

**影响**: 数据库泄露 → API Key 泄露 → 完全管理权限。

**修复**: 加密存储敏感配置，API Key 验证使用单向哈希比较。

---

## 2. 高危漏洞 (CVSS 7-8.9)

### H-01: 登录无速率限制 (CVSS 7.5)

**位置**: `web/api/public/login.go:28-81`

```go
func Login(c *gin.Context) {
    // ... 没有任何速率限制 ...
    uuid, success := accounts.CheckPassword(data.Username, data.Password)
    if !success {
        api.RespondError(c, http.StatusUnauthorized, "Invalid credentials")
        return
    }
```

**问题**: 登录接口没有任何 IP 级别或用户级别的速率限制，可进行无限暴力破解。

**影响**: 结合弱密码哈希 (C-01)，攻击者可快速爆破密码。

**修复**: 实现 IP 级别速率限制 (如每分钟 5 次尝试)，账户锁定机制。

### H-02: QQ OAuth 状态校验绕过 (CVSS 7.4)

**位置**: `web/api/public/oauth.go:29-45`

```go
providersSkipStateCheck := []string{"qq"}
if slices.Contains(providersSkipStateCheck, providerName) {
    if state == "" {
        c.JSON(400, gin.H{"status": "error", "error": "Invalid state"})
        return
    }
} else {
    if state == "" || state != c.Query("state") {
        // ...
    }
}
```

**问题**: QQ 登录忽略 OAuth state 参数的完整校验，仅检查非空。存在 CSRF 攻击风险。

**影响**: 攻击者可绑定自己的 OAuth 账号到受害者的管理员账户，获得持久访问权限。

**修复**: 对所有 OAuth 提供者使用完整的 state 校验。

### H-03: 初始管理员密码输出到日志 (CVSS 7.3)

**位置**: `database/accounts/accounts.go:65-86`, `cmd/server.go:106-112`

```go
user, passwd, err := accounts.CreateDefaultAdminAccount()
log.Println("Default admin account created. Username:", user, ", Password:", passwd)
```

**问题**: 首次运行自动生成的管理员密码通过 `log.Println` 写入标准输出，被 Docker、systemd、日志收集系统永久记录。

**影响**: 任何可以访问日志的人都能获取管理员密码。

**修复**: 强制首次登录修改密码，不在日志中输出密码，改为打印设置 URL。

### H-04: Agent Token 在 URL 查询参数中传输 (CVSS 7.2)

**位置**: `web/api/Auth.go:116-119`

```go
func extractClientToken(c *gin.Context) string {
    token := c.Query("token")  // 从 URL 参数获取
```

**问题**: Agent 认证 Token 通过 URL 查询参数 (`?token=xxx`) 传递。

**影响**: Token 泄露在浏览器历史、服务器访问日志、Referer 头中。

**修复**: 仅在 `Authorization: Bearer <token>` 头中接受 Token。

### H-05: 默认 Docker 数据库凭证 (CVSS 7.0)

**位置**: `Dockerfile:26-31`

```dockerfile
ENV KOMARI_DB_USER=root
ENV KOMARI_DB_PASS=
```

**问题**: Docker 镜像设置了默认数据库用户 root 和空密码。

**影响**: 如果开放 MySQL/PostgreSQL 端口或 SQLite 文件可被访问，数据库可被轻易入侵。

**修复**: 移除默认凭证，要求显式配置。

### H-06: API Key 绕过 2FA 敏感操作 (CVSS 7.5)

**位置**: `web/api/AuthSensitive.go:14-17`

```go
func VerifySensitive2FA(c *gin.Context) error {
    if _, ok := c.Get("api_key"); ok {
        return nil  // API Key 绕过 2FA 验证
    }
```

**问题**: 使用 API Key 进行敏感操作 (远程执行、终端访问、禁用 2FA) 时无需 2FA 验证。

**影响**: API Key 泄露后，攻击者无需 2FA 即可在所有 Agent 上执行任意命令。

**修复**: 无论使用何种认证方式，敏感操作都应要求 2FA。

---

## 3. 中危漏洞 (CVSS 4-6.9)

### M-01: 缺少安全头 (CVSS 5.3)

**位置**: `cmd/server.go`, `web/security/cors.go`

**问题**: 未设置 CSP、X-Frame-Options、X-Content-Type-Options、HSTS 等安全头。

**影响**: 存在点击劫持、MIME 嗅探攻击、协议降级攻击风险。

**修复**: 添加 HSTS (preload)、X-Frame-Options: DENY、X-Content-Type-Options: nosniff、严格 CSP。

### M-02: Session Cookie Secure 属性依赖协议检测 (CVSS 5.3)

**位置**: `web/api/public/login.go:22-30`

```go
Secure: utils.GetScheme(c) == "https",
```

**问题**: Cookie 的 Secure 标志依赖于请求协议检测。如果 `X-Forwarded-Proto` 可被伪造 (如反向代理配置不当)，Cookie 可能通过 HTTP 传输。

**影响**: Session Token 可能通过明文传输被窃取。

**修复**: 始终设置 `Secure: true`。

### M-03: 错误信息泄露内部细节 (CVSS 4.8)

**位置**: 多处 RPC handler

```go
return nil, rpc.MakeError(rpc.InternalError, "Failed to retrieve tasks: "+err.Error(), nil)
```

**问题**: 错误信息包含完整的 err.Error()，可能泄露数据库结构、文件路径等内部信息。

**影响**: 未授权用户可获取服务器内部实现细节。

**修复**: 详细错误记录到服务端日志，返回通用错误信息给客户端。

### M-04: Session 删除后未立即失效 (CVSS 5.0)

**位置**: `database/accounts/sessions.go:60-65`

**问题**: Session 从数据库删除后，已下发的 Cookie 仍可继续使用直到过期。

**影响**: 账号登出后，已泄露的 Session Token 仍可继续使用。

**修复**: 实现 Session 黑名单机制或短 TTL + 滑动刷新。

### M-05: Session 无设备指纹验证 (CVSS 4.5)

**位置**: `web/api/Auth.go:60-63`

**问题**: 虽然记录了 UserAgent 和 IP，但未用于验证 Session 合法性。

**影响**: 被盗的 Session Cookie 可从任意 IP/设备使用。

**修复**: 验证 IP 和 UserAgent 是否与创建时一致。

### M-06: Docker 默认监听 0.0.0.0 (CVSS 4.0)

**位置**: `Dockerfile:33`

```dockerfile
ENV KOMARI_LISTEN=0.0.0.0:25774
```

**问题**: 默认监听所有网络接口。

**影响**: 在容器化环境中，端口可能被意外暴露到公网。

**修复**: 默认监听 `127.0.0.1` 或明确文档说明安全风险。

### M-07: WebSocket Origin 检查可被环境变量绕过 (CVSS 3.5)

**位置**: `web/api/WebSocket.go:41-43`

```go
if strings.EqualFold(os.Getenv("KOMARI_WS_DISABLE_ORIGIN"), "true") {
    return true
}
```

**问题**: 环境变量 `KOMARI_WS_DISABLE_ORIGIN=true` 可完全关闭 WebSocket 的 Origin 校验。

**影响**: 任意网站可通过 WebSocket 与 Komari 服务器通信。

**修复**: 移除该后门或要求管理员 UI 确认。

---

## 4. 低危漏洞 (CVSS 1-3.9)

### L-01: 无密码复杂度要求 (CVSS 3.5)

**位置**: `database/accounts/accounts.go:77`

**问题**: 密码可以是任意长度和内容。

**修复**: 强制最小长度 8 位，包含大小写字母和数字。

### L-02: OAuth 绑定 Cookie 设置不安全 (CVSS 3.7)

**位置**: `web/api/public/oauth.go:73-74`

```go
c.SetCookie("binding_external_account", "", -1, "/", "", false, true)
```

**问题**: `binding_external_account` Cookie 未设置 Secure、HttpOnly 和 SameSite，`false, true` 表示 HttpOnly=false。

**修复**: 设置 `HttpOnly: true`, `Secure: true`, `SameSite: Strict`。

### L-03: 2FA Issuer 硬编码 (CVSS 2.0)

**位置**: `database/accounts/2fa.go:15`

```go
var TwoFactorIssuer = "Komari Monitor"
```

**修复**: 改为可配置项。

### L-04: Session Cookie MaxAge 硬编码 (CVSS 2.0)

**位置**: `web/api/public/login.go:15`

```go
const sessionCookieMaxAge = 2592000  // 固定 30 天
```

**修复**: 改为可配置项，允许管理员设置过期时间。

### L-05: 默认管理员用户名为 admin (CVSS 3.0)

**位置**: `database/accounts/accounts.go:71`

```go
if username == "" {
    username = "admin"
}
```

**修复**: 强制设置唯一的管理员用户名。

### L-06: 登出未通知其他活跃会话 (CVSS 2.5)

**位置**: `web/api/public/login.go:85-89`

**修复**: Session 删除时通知其他设备使 Token 失效。

### L-07: CORS 允许携带凭证 (CVSS 3.0)

**位置**: `web/security/cors.go:50-53`

```go
c.Header("Access-Control-Allow-Credentials", "true")
```

**影响**: 如果允许的 Origin 被攻破，Cookie 可能被窃取。

---

## 5. 与 Nezha 漏洞对比

| Nezha 已知漏洞 | Komari 是否受影响 | 说明 |
|----------------|-------------------|------|
| **gRPC 认证绕过** | **是 (部分)** | Nezha 兼容层存在相同问题 |
| **SSRF** | **是** | JavaScript 通知引擎可访问内部网络 |
| **SQL 注入** | **否** | GORM 参数化查询提供了保护 |
| **弱 JWT Secret** | **N/A** | Komari 使用 Session Token，非 JWT |
| **Agent 注册提权** | **是 (部分)** | Token 在 URL 中传输，Agent 认证不严格 |
| **默认凭证** | **是** | 自动生成密码被记录到日志 |
| **弱密码哈希** | **更严重** | 固定盐比 Nezha 的每用户盐更差 |
| **无速率限制** | **是** | 登录接口无限制 |
| **Session 劫持** | **部分缓解** | 设置了 HttpOnly 和 SameSite，但无设备绑定 |

**总结**: Komari 继承了 Nezha 的大部分安全问题，且密码哈希的固定盐问题比 Nezha 更严重。

---

## 6. 安全加固建议

### 立即修复 (严重级别)

| 优先级 | 问题 | 修复方案 |
|--------|------|----------|
| **P0** | C-01 弱密码哈希 | 替换为 bcrypt (cost>=12) 或 argon2id |
| **P0** | C-03 JS SSRF | 添加出站 HTTP 白名单 |
| **P0** | C-04 API Key 明文 | 加密存储或单向哈希校验 |
| **P0** | H-01 无速率限制 | 登录 IP 限流 + 账户锁定 |
| **P0** | H-03 密码输出日志 | 移除密码日志输出 |
| **P0** | H-04 Token 在 URL 中 | 迁移到 Authorization Header |

### 短期修复 (高危级别)

| 优先级 | 问题 | 修复方案 |
|--------|------|----------|
| **P1** | H-02 OAuth 绕过 | 完善所有 OAuth 提供者的 state 校验 |
| **P1** | H-06 API Key 绕过 2FA | 敏感操作始终需要 2FA |
| **P1** | M-02 Cookie Secure | 始终设置 Secure: true |
| **P1** | M-07 WS Origin 后门 | 移除环境变量绕过 |
| **P1** | M-01 缺少安全头 | 添加 HSTS/CSP/X-Frame-Options |
| **P1** | M-05 无设备指纹 | Session 绑定 IP 和 UserAgent |

### 中期修复 (中危级别)

| 优先级 | 问题 | 修复方案 |
|--------|------|----------|
| **P2** | M-03 错误信息泄露 | 通用错误信息 + 服务端日志 |
| **P2** | M-04 Session 失效延迟 | Session 黑名单或短 TTL |
| **P2** | H-05 默认 Docker 凭证 | 移除默认凭证 |
| **P2** | M-06 默认 0.0.0.0 | 默认 127.0.0.1 |
| **P2** | L-01 密码强度 | 最小长度 8 + 复杂度要求 |
| **P2** | L-02 Cookie 设置 | 设置 Secure/HttpOnly/SameSite |

### 长期建议

1. **实现完整 RBAC** - 超越 guest/client/admin 的精细权限控制
2. **数据库加密** - SQLite 文件级加密
3. **审计日志完善** - 记录所有失败登录和敏感操作
4. **CSP + Nonce** - 防止 CustomHead/CustomBody 的存储型 XSS
5. **Neza 兼容层隔离或移除** - 减少攻击面
6. **Mutual TLS** - Agent 和服务端之间的双向 TLS 认证
7. **定期安全审计** - 使用 goSec 等静态分析工具

---

## 附录: 安全分析工具

- **静态分析**: `golangci-lint`, `gosec`, `semgrep`
- **依赖扫描**: `govulncheck` (Go 官方), `snyk`
- **SAST**: `CodeQL` (GitHub)
- **DAST**: `OWASP ZAP`
- **Fuzzing**: Go 原生 Fuzzing (`go test -fuzz`)

---

*本报告由 AI 辅助生成，建议由安全团队验证后执行修复*
