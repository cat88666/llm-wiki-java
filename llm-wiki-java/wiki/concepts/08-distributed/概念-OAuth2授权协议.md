---
type: concept
status: active
name: "OAuth2授权协议"
layer: L7
aliases: ["OAuth2", "OAuth 2.0", "开放授权", "Access Token", "授权服务器", "单点登录", "SSO"]
related:
  - "[[概念-网络安全]]"
  - "[[机制-SpringMVC]]"
sources:
  - "../../../raw/note/Hollis/分布式/✅什么是OAuth2？有什么用？.md"
created: 2026-05-08
updated: 2026-05-08
lint_notes: ""
---

# OAuth2授权协议

> OAuth 2.0 是一种开放授权协议，允许用户将有限的资源访问权限授权给第三方应用，无需向其暴露自己的密码——"授权而不认证"是其核心思想。

## 第一性原理

在 OAuth2 出现前，第三方应用访问用户数据的唯一手段是用户直接提供账户密码。这带来三个问题：密码泄露风险高；无法限制第三方只访问特定范围数据；用户无法撤销单个应用的访问权限（只能改密码）。

OAuth2 通过引入**令牌（Access Token）作为临时授权凭证**，将"用户向服务商登录"和"第三方应用访问数据"两件事解耦——用户只向授权服务器证明身份，第三方拿到的是受限的、可撤销的令牌，而不是密码。

## 核心机制

### 三个角色

| 角色 | 描述 | 典型示例 |
|------|------|----------|
| **客户端（Client）** | 希望访问用户资源的第三方应用 | 浏览器 App、移动 App |
| **资源服务器（Resource Server）** | 存储用户资源的后端服务（API） | 后端接口 |
| **授权服务器（OAuth Server）** | 验证用户身份并颁发令牌 | 微信、Google、自建 OAuth |

### 授权流程（授权码模式，最常用）

```
用户 → Client：发起请求
  ↓
Client → OAuth Server：跳转授权页（携带 client_id + scope + redirect_uri）
  ↓
用户 → OAuth Server：登录并授权
  ↓
OAuth Server → Client：redirect_uri 回调，携带授权码（code）
  ↓
Client → OAuth Server：用 code + client_secret 换取 Access Token（后端对后端，不暴露给浏览器）
  ↓
Client → Resource Server：请求携带 Access Token（Bearer Token in Header）
  ↓
Resource Server：验证 Token 有效性 → 返回资源
```

**Access Token 特性**：
- 短期有效（通常 1~2 小时），可配合 Refresh Token 续期
- Scope 限制访问范围（如 `read:email`，不能读取联系人）
- Resource Server 可本地验证（JWT）或请求 OAuth Server 验证（Introspection）

### 四种授权类型

| 类型 | 适用场景 | 特点 |
|------|----------|------|
| **授权码（Authorization Code）** | Web 应用、移动 App | 最安全，code 换 token 在后端完成 |
| **隐式（Implicit）** | 纯前端 SPA（已不推荐） | Token 直接返回前端，安全性差 |
| **密码（Resource Owner Password）** | 高度信任的内部应用 | 用户将密码交给 Client，有风险 |
| **客户端凭证（Client Credentials）** | 服务间调用（无用户） | Client 用自身 ID+Secret 换 Token |

### OAuth2 与单点登录（SSO）

OAuth2 本身是**授权**协议，不是认证协议。SSO 通常在 OAuth2 基础上叠加 **OpenID Connect（OIDC）** 实现——OIDC 在 Access Token 之外增加 **ID Token（JWT 格式）**，携带用户身份信息（sub、email 等），Client 可从 ID Token 直接得知"当前是谁在登录"。

## 关键权衡

1. **令牌安全**：Access Token 一旦泄露可被滥用，必须配合 HTTPS + 短有效期 + Refresh Token 轮转机制
2. **授权码 PKCE 扩展**：移动 App 无法安全存储 client_secret，用 PKCE（Proof Key for Code Exchange）替代，防止授权码被劫持
3. **Token 验证方式**：JWT 本地验证（无需网络，但无法即时撤销）vs Introspection 远程验证（可撤销，但有网络开销）——撤销需求高选 Introspection
4. **复杂度**：OAuth2 流程和令牌管理机制较复杂，错误配置（如未验证 state 参数、redirect_uri 未校验）会引入 CSRF 和重定向攻击风险

## 与其他概念的关系

- 与 [[概念-网络安全]] 相关：OAuth2 是 API 安全的核心授权机制，Token 通过 HTTPS 传输，防止中间人攻击
- 依赖 [[机制-SpringMVC]]：Spring Security OAuth2 的 Token 验证通过 Filter/Interceptor 在请求进入 Controller 前完成

## 应用边界

**适合 OAuth2**：第三方应用集成（"用微信登录"）；微服务间服务调用授权（Client Credentials）；开放 API 平台的权限管理。

**不适合 OAuth2**：同一组织内部简单的用户认证（用 Session + JWT 即可，OAuth2 引入不必要的复杂度）；完全离线的本地系统。
