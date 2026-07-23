# Clavis-IAM 🔑
*A lightweight multi-tenant OAuth2/OIDC Identity Provider — Spring Boot 3 + Spring Security 6*

---

## 1. Final Stack

| Layer | Choice | Why |
|---|---|---|
| Language/runtime | Java 21, Spring Boot 3.3+ | current LTS, matches modern job reqs |
| Protocol engine | `spring-boot-starter-oauth2-authorization-server` | gives you `/authorize`, `/token`, `/jwks`, `/.well-known/openid-configuration`, PKCE validation for free — you don't hand-roll a security-critical spec |
| Admin API protection | `spring-boot-starter-oauth2-resource-server` | your own admin endpoints are protected by tokens issued by your own IdP — "eating your own dog food," good interview story |
| DB | PostgreSQL + Spring Data JPA | standard, works with everything else on your resume |
| Migrations | Flyway | simpler than Liquibase, same signal |
| Cache/state | Redis (Spring Data Redis) | refresh-token family store, JTI blacklist, session registry, login rate limiting |
| UI | Thymeleaf, plain CSS | only 2 pages needed: login + consent. No admin UI — REST + Swagger instead |
| Password hashing | BCrypt (Spring Security default) | simplest correct choice; swapping to Argon2 later is a 1-line change if you want the extra resume bullet |
| Mapping | MapStruct | entity ↔ DTO, avoids hand-written boilerplate mappers |
| API docs | springdoc-openapi | Swagger UI for the admin API |
| Deployment | Docker Compose (app + Postgres + Redis) | one command to run the whole thing |
| Testing | JUnit 5 + Testcontainers | used selectively — refresh rotation and realm isolation are the two things worth testing well |

**Dropped from the original PDF to keep this buildable in a real timeframe:** TOTP MFA, Kafka audit pipeline, Tailwind styling, Groups, Argon2-by-default. All parked as **Phase 5 / future work** — listing things you deliberately scoped out is itself a good signal in interviews.

**The one architectural decision to be able to explain out loud:** Spring Authorization Server owns the *protocol* (you'd be reinventing a security-critical RFC implementation for no benefit). You own the *product*: realms (multi-tenancy), RBAC, and refresh token rotation with reuse detection — the parts that are both safe to build yourself and actually differentiate the project.

---

## 2. Package Structure

```
com.yourname.clavisiam
├── ClavisIamApplication.java
│
├── config/
│   ├── AuthorizationServerConfig.java   // registers Spring Auth Server, RSA key source, token TTLs
│   ├── ResourceServerConfig.java        // protects /api/v1/admin/** with bearer tokens
│   ├── SecurityConfig.java              // form login chain for /realms/{realm}/login
│   ├── RedisConfig.java
│   ├── OpenApiConfig.java
│   └── WebConfig.java                   // CORS, message converters
│
├── tenancy/
│   ├── RealmContext.java                // ThreadLocal holder for current realm
│   ├── RealmResolverFilter.java         // parses {realm} from path, populates RealmContext
│   └── RealmAwareEntity.java            // @MappedSuperclass with realm_id column
│
├── realm/
│   ├── domain/Realm.java
│   ├── repository/RealmRepository.java
│   ├── service/RealmService.java
│   ├── controller/RealmAdminController.java
│   └── dto/{RealmDto, CreateRealmRequest}.java
│
├── client/
│   ├── domain/{Client, ClientRedirectUri}.java
│   ├── repository/ClientRepository.java
│   ├── service/ClientService.java       // also implements Spring's RegisteredClientRepository
│   ├── controller/ClientAdminController.java
│   └── dto/{ClientDto, CreateClientRequest}.java
│
├── user/
│   ├── domain/User.java
│   ├── repository/UserRepository.java
│   ├── service/{UserService, CustomUserDetailsService}.java
│   ├── controller/{UserAdminController, RegistrationController, LoginController}.java
│   └── dto/{UserDto, RegisterRequest}.java
│
├── rbac/
│   ├── domain/{Role, UserRole}.java
│   ├── repository/RoleRepository.java
│   ├── service/{RoleService, RbacEvaluator}.java
│   └── controller/RoleAdminController.java
│
├── token/
│   ├── JwtClaimsCustomizer.java         // injects realm + roles into access/ID tokens
│   ├── domain/AuthorizationEntity.java  // your own OAuth2Authorization persistence model
│   ├── repository/AuthorizationRepository.java
│   ├── service/JpaOAuth2AuthorizationService.java  // implements Spring's OAuth2AuthorizationService
│   ├── service/RefreshTokenRotationService.java     // family id + reuse detection
│   └── service/TokenBlacklistService.java           // Redis JTI blacklist, early revocation
│
├── session/
│   └── service/SessionTrackingService.java  // Redis-backed active session registry
│
├── ratelimit/
│   └── filter/LoginRateLimiterFilter.java   // Redis token-bucket on /login
│
├── audit/
│   ├── domain/AuditEvent.java
│   ├── repository/AuditEventRepository.java
│   └── service/AuditService.java            // @Async DB write; Kafka swap is Phase 5
│
├── exception/
│   ├── GlobalExceptionHandler.java          // @RestControllerAdvice
│   ├── {RealmNotFoundException, ClientNotFoundException,
│   │    UserNotFoundException, InvalidGrantException,
│   │    TokenReuseDetectedException}.java
│   └── ErrorResponse.java
│
├── mapper/
│   ├── {RealmMapper, ClientMapper, UserMapper, RoleMapper}.java   // MapStruct interfaces
│
└── util/
    ├── PasswordUtil.java
    ├── JwkUtil.java              // RSA keypair generation/loading
    └── IdGenerator.java          // UUID/ULID helpers
```

---

## 3. Key Entities

| Entity | Key fields | Notes |
|---|---|---|
| `Realm` | id, name, displayName, enabled, accessTokenTtl, refreshTokenTtl | tenant boundary; everything else has a `realm_id` FK |
| `Client` | id, realmId, clientId, clientSecretHash, redirectUris, grantTypes, scopes | wraps into Spring's `RegisteredClient` at runtime |
| `ClientRedirectUri` | id, clientId, uri | one-to-many, own table since clients can have multiple |
| `User` | id, realmId, username, email, passwordHash, enabled, emailVerified | unique constraint on (realmId, username) — not globally unique |
| `Role` | id, realmId, name, description | realm-scoped, e.g. `admin`, `viewer` |
| `UserRole` | userId, roleId | join table |
| `AuthorizationEntity` | id, registeredClientId, principalName, authorizationGrantType, accessTokenValue, accessTokenExpiresAt, refreshTokenValue, refreshTokenFamilyId, revoked | backs Spring's `OAuth2AuthorizationService` — this is where rotation state lives |
| `AuditEvent` | id, realmId, eventType, userId, clientId, ip, timestamp, detailsJson | login, token issue/revoke, role change |

**Multi-tenancy pattern:** every realm-scoped table carries `realm_id`, and every repository query is implicitly filtered by `RealmContext.current()` (via a Hibernate filter or a base repository method — pick one, Hibernate `@FilterDef` is the more "I understand the ORM" answer in an interview).

---

## 4. Configs

- **`AuthorizationServerConfig`** — `SecurityFilterChain` for the `/oauth2/**` endpoints, RSA `JWKSource`, token settings (TTLs pulled per-realm from `RealmContext`), wires your `JpaOAuth2AuthorizationService` in place of the default in-memory one.
- **`ResourceServerConfig`** — separate filter chain, `@Order` before the auth server chain, validates bearer tokens on `/api/v1/admin/**` against your own JWKS.
- **`SecurityConfig`** — form login chain for `/realms/{realm}/login`, wires `CustomUserDetailsService` + `RealmResolverFilter`.
- **`RedisConfig`** — connection factory, `RedisTemplate<String,String>` bean.
- **`OpenApiConfig`** — Swagger UI grouping for admin endpoints only (protocol endpoints are self-documenting via `.well-known`).

---

## 5. Controllers

| Controller | Endpoints | Auth |
|---|---|---|
| *(none — Spring provides these)* | `GET/POST /oauth2/authorize`, `POST /oauth2/token`, `GET /oauth2/jwks`, `GET /.well-known/openid-configuration` | handled by the starter |
| `RegistrationController` | `POST /realms/{realm}/register` | public |
| `LoginController` | `GET /realms/{realm}/login` (Thymeleaf view) | public |
| `RealmAdminController` | CRUD `/api/v1/admin/realms` | bearer token, `admin` role |
| `ClientAdminController` | CRUD `/api/v1/admin/realms/{realm}/clients` | bearer token, `admin` role |
| `UserAdminController` | CRUD `/api/v1/admin/realms/{realm}/users` | bearer token, `admin` role |
| `RoleAdminController` | CRUD `/api/v1/admin/realms/{realm}/roles`, role assignment | bearer token, `admin` role |

Note: OIDC `/userinfo` isn't automatic — you configure it via `OidcUserInfoEndpointConfigurer` in `AuthorizationServerConfig`, mapping claims from your `User` entity. Worth knowing this is a manual step, it comes up in interviews.

---

## 6. Services

| Service | Responsibility |
|---|---|
| `RealmService` | CRUD + realm-enabled checks |
| `ClientService` | CRUD + implements `RegisteredClientRepository` so Spring's engine can look up clients from your DB |
| `UserService` / `CustomUserDetailsService` | user CRUD; `loadUserByUsername` scoped to `RealmContext.current()` |
| `RoleService` / `RbacEvaluator` | role CRUD, resolves a user's effective roles into JWT claims |
| `JpaOAuth2AuthorizationService` | implements Spring's `OAuth2AuthorizationService` — persists/loads authorizations from `AuthorizationEntity` instead of the default in-memory store |
| `RefreshTokenRotationService` | on every refresh: issue new token, mark old one used, **if an already-used token is presented again, revoke the entire `refreshTokenFamilyId`** — this is your reuse-detection story |
| `TokenBlacklistService` | Redis `SET jti:{id} EX {ttl}` on logout/admin-revoke; checked in the resource-server filter chain for early revocation despite JWT statelessness |
| `SessionTrackingService` | Redis-backed "active sessions per user" for the admin `Sessions` view (mirrors the Keycloak screenshot you started from) |
| `AuditService` | `@Async` write to `AuditEvent` table on login/logout/token events |

---

## 7. Filters

| Filter | Order | Purpose |
|---|---|---|
| `RealmResolverFilter` | first | extracts `{realm}` from the path, loads `Realm`, populates `RealmContext`, 404s if disabled/missing |
| `LoginRateLimiterFilter` | before auth | Redis token-bucket per (realm, username or IP) on `/login` and `/oauth2/token` password grant attempts |

Everything else (JWT validation on admin APIs) is handled by `spring-boot-starter-oauth2-resource-server`'s built-in filter — no need to hand-roll a JWT filter.

---

## 8. Exceptions

`GlobalExceptionHandler` (`@RestControllerAdvice`) maps each to an HTTP status + `ErrorResponse` body:

- `RealmNotFoundException` → 404
- `ClientNotFoundException` → 404
- `UserNotFoundException` → 404
- `InvalidGrantException` → 400 (OAuth2-spec-shaped error body: `error`, `error_description`)
- `TokenReuseDetectedException` → 401 + triggers full family revocation as a side effect before responding

---

## 9. Mappers (MapStruct)

`RealmMapper`, `ClientMapper`, `UserMapper`, `RoleMapper` — each a simple `@Mapper(componentModel = "spring")` interface, entity ↔ DTO. Keeps controllers thin and stops password hashes / internal IDs leaking into API responses.

---

## 10. Repositories

Standard `JpaRepository<T, UUID>` for each entity, plus:
- `ClientRepository.findByRealmIdAndClientId(...)`
- `UserRepository.findByRealmIdAndUsername(...)`
- `AuthorizationRepository.findByRefreshTokenFamilyId(...)` — needed for the rotation/reuse logic

---

## 11. Util

- `PasswordUtil` — thin wrapper over `BCryptPasswordEncoder`
- `JwkUtil` — generates/loads the RSA keypair at startup (or from a mounted secret in Docker), builds the `JWKSource`
- `IdGenerator` — UUID/ULID helper for entity IDs

---

## 12. What Spring Gives You Free vs. What You Build

| Concern | Free (starter) | You build |
|---|---|---|
| Authorization Code flow | ✅ | — |
| PKCE validation | ✅ | — |
| Token endpoint, grant handling | ✅ | — |
| JWKS + discovery endpoints | ✅ | — |
| Client storage | interface only | `ClientService implements RegisteredClientRepository` |
| Authorization/token storage | interface only | `JpaOAuth2AuthorizationService` |
| **Multi-tenancy (realms)** | ❌ | `RealmContext`, `RealmResolverFilter`, realm-scoped queries |
| **Refresh token rotation + reuse detection** | basic rotation only | family-id tracking, reuse → full family revocation |
| **RBAC** | ❌ | `Role`, `UserRole`, `RbacEvaluator`, claims customization |
| Early token revocation | ❌ (JWTs are stateless by default) | Redis JTI blacklist checked in resource-server chain |
| Rate limiting | ❌ | Redis token-bucket filter |

---

## 13. Build Roadmap

**Phase 1 — Protocol core**
Spring Boot + Authorization Server starter wired to an in-memory client first (sanity check), then swap to `ClientService`/`JpaOAuth2AuthorizationService` backed by Postgres. Flyway schema for Realm/Client/User.

**Phase 2 — Multi-tenancy**
`RealmResolverFilter` + `RealmContext`, realm-scoped user lookup, per-realm token TTLs, login page reflects the realm.

**Phase 3 — RBAC + refresh rotation**
Role/UserRole model, `RbacEvaluator` injecting roles into JWT claims via `JwtClaimsCustomizer`. `RefreshTokenRotationService` with family-id reuse detection — this is the phase to slow down on, it's your best interview material.

**Phase 4 — Hardening**
Redis JTI blacklist for logout/admin revoke, login rate limiter, audit log, Swagger for admin API, Dockerize (app + Postgres + Redis via Compose).

**Phase 5 — Optional stretch**
TOTP MFA, Argon2 swap, Kafka-based audit pipeline (reuses your plagiarism-detector Kafka experience), one social login (GitHub) to prove you understand OIDC as a *client* too, not just a provider.

---
