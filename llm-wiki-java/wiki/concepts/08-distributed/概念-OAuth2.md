---
type: concept
status: active
name: "OAuth2授权协议"
layer: L7
aliases: ["OAuth2", "OAuth 2.0", "开放授权", "Access Token", "授权服务器", "单点登录", "SSO", "OIDC", "PKCE", "Refresh Token", "授权码模式"]
tags: ["#distributed"]
related:
  - "[[概念-网络安全]]"
  - "[[机制-SpringMVC]]"
sources:
  - "../../../raw/note/Hollis/分布式/✅什么是OAuth2？有什么用？.md"
created: 2026-05-08
updated: 2026-05-15
lint_notes: ""
---

# OAuth2授权协议

> OAuth 2.0 是一种开放授权协议，允许用户将有限的资源访问权限授权给第三方应用，无需向其暴露自己的密码——"授权而不认证"是其核心思想。

## 快速导航

| 标题索引 | 概述 |
| --- | --- |
| [一、第一性原理](#一第一性原理) | 密码泄露问题、令牌作为中间层 |
| [二、三个核心角色](#二三个核心角色) | Client/Resource Server/OAuth Server |
| [三、授权码模式流程](#三授权码模式流程) | 最安全的标准流程（code → token） |
| [四、四种授权类型](#四四种授权类型) | 授权码/隐式/密码/客户端凭证 |
| [五、OAuth2 与 SSO](#五oauth2-与-sso) | OIDC = OAuth2 + 身份认证 + ID Token |
| [六、PKCE 扩展](#六pkce-扩展) | 移动 App 无法安全存储 client_secret 的解决方案 |
| [七、Token 验证方式](#七token-验证方式) | JWT 本地验证 vs Introspection 远程验证 |
| [八、关键权衡](#八关键权衡) | 安全性、复杂度、可撤销性 |
| [九、与其他概念的关系](#九与其他概念的关系) | 网络安全、SpringMVC |
| [十、应用边界](#十应用边界) | 适合 vs 不适合场景 |

## 一、第一性原理

在 OAuth2 出现前，第三方应用访问用户数据的唯一手段是用户直接提供账户密码。这带来三个问题：
1. **密码泄露风险高**：用户把密码给第三方，密码对等于全权授权
2. **无法限制访问范围**：无法只允许第三方读取邮件而不能操作联系人
3. **无法撤销单个应用**：只能改密码来撤销所有应用的权限

OAuth2 通过引入**令牌（Access Token）作为临时授权凭证**，将"用户向服务商登录"和"第三方应用访问数据"两件事解耦——用户只向授权服务器证明身份，第三方拿到的是受限的、可撤销的令牌，而不是密码。

## 二、三个核心角色

| 角色 | 描述 | 典型示例 |
|------|------|----------|
| **客户端（Client）** | 希望访问用户资源的第三方应用 | 浏览器 App、移动 App |
| **资源服务器（Resource Server）** | 存储用户资源的后端服务（API） | 后端接口 |
| **授权服务器（OAuth Server）** | 验证用户身份并颁发令牌 | 微信、Google、自建 OAuth |

## 三、授权码模式流程

授权码模式（Authorization Code）是最安全、最常用的模式：

```
用户 → Client：发起请求
  ↓
Client → OAuth Server：跳转授权页（携带 client_id + scope + redirect_uri + state）
  ↓
用户 → OAuth Server：登录并授权
  ↓
OAuth Server → Client：redirect_uri 回调，携带授权码（code）
  ↓
Client 后端 → OAuth Server：用 code + client_secret 换取 Access Token
  （后端对后端，不暴露给浏览器）
  ↓
Client → Resource Server：请求携带 Access Token（Bearer Token in Header）
  ↓
Resource Server：验证 Token 有效性 → 返回资源
```

**state 参数**：防 CSRF 攻击，每次请求随机生成，回调时必须校验。

**Access Token 特性**：
- 短期有效（通常 1~2 小时），可配合 Refresh Token 续期
- Scope 限制访问范围（如 `read:email`，不能读取联系人）
- Resource Server 可本地验证（JWT）或请求 OAuth Server 验证（Introspection）

## 四、四种授权类型

| 类型 | 适用场景 | 特点 |
|------|----------|------|
| **授权码（Authorization Code）** | Web 应用、移动 App | 最安全，code 换 token 在后端完成 |
| **隐式（Implicit）** | 纯前端 SPA（已不推荐）| Token 直接返回前端，安全性差 |
| **密码（Resource Owner Password）** | 高度信任的内部应用 | 用户将密码交给 Client，有风险 |
| **客户端凭证（Client Credentials）** | 服务间调用（无用户）| Client 用自身 ID+Secret 换 Token |

**微服务间调用**用 Client Credentials 模式：服务 A 以自身身份（client_id + client_secret）向 OAuth Server 申请 Token，调用服务 B 时携带此 Token，服务 B 验证后才放行。

## 五、OAuth2 与 SSO

OAuth2 本身是**授权**协议，不是认证协议。SSO 通常在 OAuth2 基础上叠加 **OpenID Connect（OIDC）** 实现：

- **OIDC** 在 Access Token 之外增加 **ID Token（JWT 格式）**，携带用户身份信息（sub、email、name 等）
- Client 可从 ID Token 直接得知"当前是谁在登录"，无需再次查询用户信息
- SSO 流程：用户在认证中心登录 → 生成全局 Session + ID Token → 其他子系统信任 ID Token

## 六、PKCE 扩展

**问题**：移动 App（iOS/Android）和 SPA 无法安全存储 `client_secret`（代码可被反编译），若使用标准授权码模式，`client_secret` 泄露则安全机制完全失效。

**PKCE（Proof Key for Code Exchange）解法**：
```
Client 生成 code_verifier（随机字符串）
→ code_challenge = SHA256(code_verifier)（单向哈希）
→ 发送授权请求时附带 code_challenge
→ 换 Token 时发送 code_verifier，OAuth Server 验证其哈希值与 code_challenge 匹配
```

即使授权码被截获，攻击者没有 code_verifier 也无法换取 Token。

## 七、Token 验证方式

| 方式 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **JWT 本地验证** | Resource Server 用公钥本地验签 | 无网络开销，性能好 | Token 被吊销后无法即时失效（直到自然过期）|
| **Introspection 远程验证** | Resource Server 每次请求 OAuth Server 验证 | 可即时撤销 Token | 每次请求多一跳网络开销 |

**最佳实践**：Access Token 短期有效（1小时）+ JWT 本地验证；Refresh Token 长期有效 + Introspection 或 DB 查询验证（因为 Refresh Token 需要支持撤销）。

## 八、关键权衡

1. **令牌安全**：Access Token 一旦泄露可被滥用，必须配合 HTTPS + 短有效期 + Refresh Token 轮转机制
2. **JWT vs Introspection**：撤销需求高（如用户注销立即失效）选 Introspection；追求性能且可接受有效期内无法撤销选 JWT
3. **PKCE 使用场景**：移动 App 和 SPA 必须使用 PKCE，Web App 后端可以安全存储 client_secret 则不需要
4. **复杂度**：OAuth2 流程和令牌管理机制较复杂，错误配置（如未验证 state 参数、redirect_uri 未校验）会引入 CSRF 和重定向攻击风险

## 九、与其他概念的关系

- 与 [[概念-网络安全]] 相关：OAuth2 是 API 安全的核心授权机制，Token 通过 HTTPS 传输，防止中间人攻击；与 JWT、HMAC 签名结合实现防篡改
- 依赖 [[机制-SpringMVC]]：Spring Security OAuth2 的 Token 验证通过 Filter/Interceptor 在请求进入 Controller 前完成；Spring Authorization Server 实现标准 OAuth2 服务端

## 十、应用边界

**适合 OAuth2**：
- 第三方应用集成（"用微信登录"、"用 Google 登录"）
- 微服务间服务调用授权（Client Credentials 模式）
- 开放 API 平台的权限管理（Scope 精细控制）
- 多系统 SSO（结合 OIDC）

**不适合 OAuth2**：
- 同一组织内部简单的用户认证（用 Session + JWT 即可，OAuth2 引入不必要的复杂度）
- 完全离线的本地系统
- 简单的 API Key 认证场景（过度设计）
