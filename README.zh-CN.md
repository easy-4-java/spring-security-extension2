# spring-security-extension2

![Java](https://img.shields.io/badge/Java-17-orange) ![License](https://img.shields.io/badge/License-Apache%202.0-blue)

[English](./README.md) | [简体中文](./README.zh-CN.md)

[1. Project Overview](#1-project-overview) | [2. Features & Status](#2-features--status) | [3. Requirements & Compatibility](#3-requirements--compatibility) | [4. Architecture & Modules](#4-architecture--modules) | [5. Installation](#5-installation) | [6. Quick Start](#6-quick-start) | [7. Configuration](#7-configuration) | [8. Core Usage / API](#8-core-usage--api) | [9. Testing & Build](#9-testing--build) | [10. Versioning & Branches](#10-versioning--branches) | [11. Contributing & License](#11-contributing--license)

## 1. Project Overview

`spring-security-extension2`（"Security Biz SDK"）是 easy4j Spring Security 生态的共享契约层，独立于 Spring Boot：认证流程使用的常量、POJO 与接口——`SecurityPrincipal` 用户模型、`JwtPayloadRepository` JWT 契约、结构化的 `AuthResponse` / `AuthResponseCode` 结果、登录/验证码模型、会话固定策略与安全头属性模型。

它承载安全相关模块之间共享的类型——本身不含过滤器、Provider 或自动配置（这些位于兄弟模块 `spring-security-extension`）。

典型场景：

| 场景 | 本模块提供的组件 |
|:---|:---|
| 类型化认证主体 | `SecurityPrincipal`（继承 Spring Security `User`，含 uid/uuid/ukey/ucode/rid/rkey/rcode 字段） |
| JWT 载荷契约 | `JwtPayloadRepository`（由 token 签发 JWT、构建 `UserProfilePayload`） |
| 统一认证响应 | `AuthResponse` / `AuthResponseCode` / `ApiCodeValue` / `AuthConstants` |
| 登录请求模型 | `PostLoginRequest`（username / password / captcha） |
| 验证码契约 | `CaptchaResolver` 接口 + `NullCaptchaResolver` |
| 安全设置模型 | `SecurityEntryPointProperties`、`SecurityRedirectProperties`、`SecurityLogoutProperties`、`SecurityRequestProperties`、`SessionFixationPolicy`、header 属性模型 |

## 2. Features & Status

项目状态：`1.0.x.*` 预发布开发线（快照版本）；在首个正式 Release 标签之前，公开 API 仍在稳定过程中。

| 能力 | 状态 | 说明 |
|:---|:---|:---|
| 主体模型 | 稳定 | `SecurityPrincipal extends User implements Cloneable`——uid、uuid、ukey、ucode、rid、rkey、rcode + `toPayload()` |
| JWT 契约 | 稳定 | `JwtPayloadRepository`——默认 `issueJwt(AbstractAuthenticationToken)` 与 `getProfilePayload(...)` 辅助方法 |
| 用户画像载荷 | 稳定 | `UserProfilePayload`——身份字段 + 已签发 token |
| 认证响应模型 | 稳定 | `AuthResponse<T>` 及 `success(...)` / `fail(...)` 工厂方法；`AuthResponseCode`、`ApiCodeValue`、`AuthConstants` |
| 登录请求模型 | 稳定 | `PostLoginRequest(username, password, captcha)` |
| 失败计数器 | 稳定 | `AuthenticatingFailureCounter` 用于登录尝试跟踪 |
| 验证码契约 | 稳定 | `CaptchaResolver` 接口 + `NullCaptchaResolver` |
| 安全设置 POJO | 稳定 | `SecurityEntryPointProperties`、`SecurityRedirectProperties`、`SecurityLogoutProperties`、`SecurityRequestProperties`、`SessionFixationPolicy`（枚举）、`property/header/*` |

## 3. Requirements & Compatibility

| 要求 | 版本 |
|:---|:---|
| JDK | 17+ |
| Maven | 3.6+ |
| Spring Security | spring-security-core（由 POM 管理） |
| easy4j 兄弟模块 | `jwt-issuer-api`（同一 `1.0.x.*` 版本线） |
| 其他运行依赖 | swagger-annotations、commons-lang3、javax.servlet-api |

版本线：

| 分支 | JDK | 版本模式 | 说明 |
|:---|:---|:---|:---|
| `feature/1.0.x` | 8 | `1.0.x.*` | 当前开发线；Spring Security 5.x 时代 |
| `feature/2.0.x` | 17 | `2.0.x.*` | 下一条版本线 |
| `feature/3.0.x` | 21 | `3.0.x.*` | 未来版本线 |

## 4. Architecture & Modules

```
Authentication flow (sibling module spring-security-extension)
        |
        v
+------------------------------------+
| JwtPayloadRepository (issueJwt,    |
|  getProfilePayload)                |
+------------------------------------+
        |
        v
+------------------------------------+
| SecurityPrincipal -> UserProfile-  |
| Payload (uid/uuid/ukey/... + token)|
+------------------------------------+
        |
        v
AuthResponse / AuthResponseCode (unified result)
```

本工程为单 jar 模块，包位于 `org.springframework.security.extension`：

| 包 | 职责 |
|:---|:---|
| `userdetails` | `SecurityPrincipal`、`UserProfilePayload`、`JwtPayloadRepository` |
| `exception` | `AuthResponse`、`AuthResponseCode`、`ApiCodeValue`、`AuthConstants` |
| `authentication` | `PostLoginRequest`、`AuthenticatingFailureCounter`、`captcha/`（`CaptchaResolver`、`NullCaptchaResolver`） |
| `property` | 安全设置 POJO + `header/` 安全头属性模型 |

## 5. Installation

制品发布到 easy4j 私有仓库与 GitHub Releases，暂未发布 Maven Central。

Maven：

```xml
<dependency>
    <groupId>io.github.easy4j</groupId>
    <artifactId>spring-security-extension2</artifactId>
    <version>2.0.x.x.20260630-SNAPSHOT</version>
</dependency>
```

Gradle：

```groovy
implementation 'io.github.easy4j:spring-security-extension2:2.0.x.x.20260630-SNAPSHOT'
```

## 6. Quick Start

创建主体并转换为携带已签发 JWT 的画像载荷：

```java
import org.springframework.security.extension.userdetails.SecurityPrincipal;
import org.springframework.security.extension.userdetails.UserProfilePayload;

SecurityPrincipal principal = new SecurityPrincipal("demo", "password", "ROLE_USER");
principal.setUid("u-1001");

// JwtPayloadRepository 实现负责签发 token：
// String jwt = jwtPayloadRepository.issueJwt(authenticationToken);

UserProfilePayload payload = principal.toPayload();
payload.setToken("eyJ...");   // 由 JwtPayloadRepository 签发的 token
System.out.println(payload.getUid()); // u-1001
```

预期结果：主体携带身份字段，`toPayload()` 产出 `UserProfilePayload`，可继续填充已签发的 JWT。

## 7. Configuration

纯 POJO 与接口库——无配置文件与属性前缀。安全设置 POJO（`SecurityEntryPointProperties`、`SecurityRedirectProperties`、`SecurityLogoutProperties`、`SecurityRequestProperties`、header 属性）由应用填充，并交由装配模块消费。

## 8. Core Usage / API

实现 JWT 契约：

```java
import org.springframework.security.extension.userdetails.JwtPayloadRepository;

JwtPayloadRepository jwtPayloadRepository = new JwtPayloadRepository() {
    @Override
    public String issueJwt(SecurityPrincipal principal) {
        // 为主体签名并返回 JWT
        return "eyJ...";
    }
};
```

返回统一认证结果：

```java
import org.springframework.security.extension.exception.AuthResponse;

AuthResponse<Object> ok = AuthResponse.success("login ok");
AuthResponse<Object> bad = AuthResponse.fail(400, "bad credentials");
```

## 9. Testing & Build

构建：

```bash
./mvnw clean verify
```

- 构建配置了 JaCoCo Maven 插件：覆盖率报告生成于 `target/site/jacoco/index.html`，并配置了 BUNDLE 行覆盖率 90% 的校验规则（`haltOnFailure=false`，即只报告不阻断构建）；
- 当前仓库本模块暂无单元测试，覆盖率以 JaCoCo 报告为准；
- `central` Maven Profile（`./mvnw -Pcentral deploy`）附加 GPG 签名、源码包与 Javadoc 包用于发布。

## 10. Versioning & Branches

维护三条并行版本线：

| 分支 | JDK | 版本模式 |
|:---|:---|:---|
| `feature/1.0.x` | 8 | `1.0.x.*` |
| `feature/2.0.x` | 17 | `2.0.x.*` |
| `feature/3.0.x` | 21 | `3.0.x.*` |

维护策略：`1.0.x` 为当前活跃开发线（当前快照 `2.0.x.x.20260630-SNAPSHOT`）；`2.0.x` 与 `3.0.x` 为面向更新 JDK 的前向移植线。快照按需构建，正式 Release 通过 GitHub Releases 分发。

## 11. Contributing & License

- Fork 仓库并提交 Pull Request；`1.0.x` 版本线保持 JDK 8 兼容；
- Bug 反馈与功能建议通过 GitHub Issues 跟踪；
- 基于 [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0) 开源。
