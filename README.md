# spring-security-extension2

[English](./README.md) | [简体中文](./README.zh-CN.md)

## Table of Contents

- [1. Project Overview](#1-project-overview)
- [2. Features & Status](#2-features--status)
- [3. Requirements & Compatibility](#3-requirements--compatibility)
- [4. Architecture & Modules](#4-architecture--modules)
- [5. Installation](#5-installation)
- [6. Quick Start](#6-quick-start)
- [7. Configuration](#7-configuration)
- [8. Core Usage / API](#8-core-usage--api)
- [9. Testing & Build](#9-testing--build)
- [10. Versioning & Branches](#10-versioning--branches)
- [11. Contributing & License](#11-contributing--license)

## 1. Project Overview

`spring-security-extension2` ("Security Biz SDK") is the shared contract layer of the easy4j Spring Security ecosystem, independent of Spring Boot: constants, POJOs and interfaces that authentication flows use — a `SecurityPrincipal` user model, the `JwtPayloadRepository` JWT contract, structured `AuthResponse` / `AuthResponseCode` results, login/captcha models, session-fixation policy and security header property models.

It holds the types shared between security-related modules — it contains no filters, providers or auto-configuration (those live in the sibling `spring-security-extension` module).

Typical scenarios:

| Scenario | What this module contributes |
|:---|:---|
| Typed authenticated principal | `SecurityPrincipal` (extends Spring Security `User`, with uid/uuid/ukey/ucode/rid/rkey/rcode fields) |
| JWT payload contract | `JwtPayloadRepository` (issue JWT from a token, build `UserProfilePayload`) |
| Unified auth responses | `AuthResponse` / `AuthResponseCode` / `ApiCodeValue` / `AuthConstants` |
| Login request model | `PostLoginRequest` (username / password / captcha) |
| Captcha contract | `CaptchaResolver` interface + `NullCaptchaResolver` |
| Security settings models | `SecurityEntryPointProperties`, `SecurityRedirectProperties`, `SecurityLogoutProperties`, `SecurityRequestProperties`, `SessionFixationPolicy`, header property models |

## 2. Features & Status

Project status: pre-release development line (`1.0.x.*` snapshots); public API is still stabilizing until the first tagged release.

| Capability | Status | Notes |
|:---|:---|:---|
| Principal model | Stable | `SecurityPrincipal extends User implements Cloneable` — uid, uuid, ukey, ucode, rid, rkey, rcode + `toPayload()` |
| JWT contract | Stable | `JwtPayloadRepository` — default `issueJwt(AbstractAuthenticationToken)` and `getProfilePayload(...)` helpers |
| User profile payload | Stable | `UserProfilePayload` — identity fields plus issued token |
| Auth response model | Stable | `AuthResponse<T>` with `success(...)` / `fail(...)` factories; `AuthResponseCode`, `ApiCodeValue`, `AuthConstants` |
| Login request model | Stable | `PostLoginRequest(username, password, captcha)` |
| Failure counter | Stable | `AuthenticatingFailureCounter` for login attempt tracking |
| Captcha contract | Stable | `CaptchaResolver` interface + `NullCaptchaResolver` |
| Security settings POJOs | Stable | `SecurityEntryPointProperties`, `SecurityRedirectProperties`, `SecurityLogoutProperties`, `SecurityRequestProperties`, `SessionFixationPolicy` (enum), `property/header/*` |

## 3. Requirements & Compatibility

| Requirement | Version |
|:---|:---|
| JDK | 21+ |
| Maven | 3.6+ |
| Spring Security | spring-security-core (managed by the POM) |
| easy4j sibling module | `jwt-issuer-api` (same `1.0.x.*` line) |
| Other runtime deps | swagger-annotations, commons-lang3, javax.servlet-api |

Version lines:

| Branch | JDK | Version pattern | Notes |
|:---|:---|:---|:---|
| `feature/1.0.x` | 8 | `1.0.x.*` | Current line; Spring Security 5.x era |
| `feature/2.0.x` | 17 | `2.0.x.*` | Next line |
| `feature/3.0.x` | 21 | `3.0.x.*` | Future line |

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

The project is a single jar module. Packages under `org.springframework.security.extension`:

| Package | Responsibility |
|:---|:---|
| `userdetails` | `SecurityPrincipal`, `UserProfilePayload`, `JwtPayloadRepository` |
| `exception` | `AuthResponse`, `AuthResponseCode`, `ApiCodeValue`, `AuthConstants` |
| `authentication` | `PostLoginRequest`, `AuthenticatingFailureCounter`, `captcha/` (`CaptchaResolver`, `NullCaptchaResolver`) |
| `property` | Security settings POJOs + `header/` security header property models |

## 5. Installation

Artifacts are published to the easy4j private repository and GitHub Releases; the project is not yet on Maven Central.

Maven:

```xml
<dependency>
    <groupId>io.github.easy4j</groupId>
    <artifactId>spring-security-extension2</artifactId>
    <version>3.0.x.x.20260630-SNAPSHOT</version>
</dependency>
```

Gradle:

```groovy
implementation 'io.github.easy4j:spring-security-extension2:3.0.x.x.20260630-SNAPSHOT'
```

## 6. Quick Start

Create a principal and turn it into a profile payload with an issued JWT:

```java
import org.springframework.security.extension.userdetails.SecurityPrincipal;
import org.springframework.security.extension.userdetails.UserProfilePayload;

SecurityPrincipal principal = new SecurityPrincipal("demo", "password", "ROLE_USER");
principal.setUid("u-1001");

// A JwtPayloadRepository implementation issues the token:
// String jwt = jwtPayloadRepository.issueJwt(authenticationToken);

UserProfilePayload payload = principal.toPayload();
payload.setToken("eyJ...");   // token issued by the JwtPayloadRepository
System.out.println(payload.getUid()); // u-1001
```

Expected result: the principal carries the identity fields, and `toPayload()` produces a `UserProfilePayload` ready to be completed with the issued JWT.

## 7. Configuration

Pure library of POJOs and interfaces — no configuration files or property prefixes. The security settings POJOs (`SecurityEntryPointProperties`, `SecurityRedirectProperties`, `SecurityLogoutProperties`, `SecurityRequestProperties`, header properties) are populated by the application and consumed by the wiring module.

## 8. Core Usage / API

Implement the JWT contract:

```java
import org.springframework.security.extension.userdetails.JwtPayloadRepository;

JwtPayloadRepository jwtPayloadRepository = new JwtPayloadRepository() {
    @Override
    public String issueJwt(SecurityPrincipal principal) {
        // sign and return the JWT for the principal
        return "eyJ...";
    }
};
```

Return unified auth results:

```java
import org.springframework.security.extension.exception.AuthResponse;

AuthResponse<Object> ok = AuthResponse.success("login ok");
AuthResponse<Object> bad = AuthResponse.fail(400, "bad credentials");
```

## 9. Testing & Build

Build:

```bash
./mvnw clean verify
```

- The build is configured with the JaCoCo Maven plugin: a coverage report is generated at `target/site/jacoco/index.html` and a rule checks the bundle line coverage against a 90% minimum (`haltOnFailure=false`, so the check reports but does not fail the build).
- The repository currently ships no unit tests for this module; coverage is tracked via the JaCoCo report.
- The `central` Maven profile (`./mvnw -Pcentral deploy`) attaches GPG signatures, sources and Javadoc jars for publishing.

## 10. Versioning & Branches

Three parallel version lines are maintained:

| Branch | JDK | Version pattern |
|:---|:---|:---|
| `feature/1.0.x` | 8 | `1.0.x.*` |
| `feature/2.0.x` | 17 | `2.0.x.*` |
| `feature/3.0.x` | 21 | `3.0.x.*` |

Maintenance policy: the `1.0.x` line is the actively developed line (current snapshot `3.0.x.x.20260630-SNAPSHOT`); `2.0.x` and `3.0.x` are forward porting lines targeting newer JDKs. Snapshots are built on demand; tagged releases are distributed via GitHub Releases.

## 11. Contributing & License

- Fork the repository and open a pull request; keep the `1.0.x` line compatible with JDK 8.
- Bug reports and feature requests are tracked via GitHub Issues.
- Licensed under the [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0).
