# Clavis-IAM 🔑

Clavis-IAM is a lightweight, monolithic Identity and Access Management (IAM) server built with **Spring Boot 3**, **Spring Security 6**, and **Spring Data JPA** backed by **PostgreSQL**. 

Designed as a clean alternative to legacy enterprise tools like Keycloak, it delivers a production-ready OAuth 2.0 and OpenID Connect (OIDC) engine optimized for modern web applications and microservices.

---

# 1. Project Overview

Clavis-IAM implements standard authorization and authentication flows cleanly within a single Spring Boot application.

## Key Capabilities

- **OAuth 2.0 & OIDC Engine:** Implements Authorization Code Flow with PKCE (RFC 7636).
- **Multi-Tenancy ("Realms"):** Logical data isolation boundaries separating users, credentials, clients, and roles per tenant.
- **Stateless Token Minting:** Issues cryptographically signed RSA Access Tokens, ID Tokens, and Refresh Tokens.
- **JWKS Key Distribution:** Exposes `/.well-known/jwks.json` for external stateless token verification.
- **Embedded UI:** Server-rendered pages (Thymeleaf + Tailwind CSS) for multi-tenant Login, Registration, and OAuth2 Consent.
- **BCrypt Credential Hashing:** Secure password hashing using `BCryptPasswordEncoder`.
- **Administrative REST API:** Full programmatic control over realms, clients, users, and roles.

---

# 2. Tech Stack

| Category | Technology |
|----------|------------|
| **Language & Framework** | Java 17+, Spring Boot 3.2+ |
| **Security** | Spring Security 6, Spring Authorization Server |
| **Persistence** | Spring Data JPA, PostgreSQL 15 |
| **Frontend UI** | Thymeleaf, Tailwind CSS |
| **Containerization** | Docker, Docker Compose |

---

# 3. Architecture & Domain Model

```text
                    [ Realm ]
         (Security Boundary per Tenant)
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
     [ Clients ]            [ Users ]
          │                     │
          │                     ├── Credentials (BCrypt)
          │                     ├── Roles
          │                     └── Refresh Tokens
          │
          ├── Redirect URIs
          ├── Client Secrets
          └── Allowed Scopes
```

## Entity Relationship Model

```text
+------------------+         +--------------------+         +-----------------------+
|      Realm       | <1---*  |       Client       | <1---*  |   ClientRedirectUri   |
+------------------+         +--------------------+         +-----------------------+
| id (UUID)        |         | id (UUID)          |         | id (UUID)             |
| name             |         | client_id          |         | uri                   |
| enabled          |         | client_secret_hash |         +-----------------------+
+------------------+         | grant_types        |
                             | scopes             |
                             +--------------------+
          │
          │ 1
          │
          └─── * +--------------------+         +-----------------------+
                 |        User        | <*---*> |         Role          |
                 +--------------------+         +-----------------------+
                 | id (UUID)          |         | id (UUID)             |
                 | username           |         | name                  |
                 | email              |         | permissions (JSON)    |
                 | password_hash      |         +-----------------------+
                 | enabled            |
                 +--------------------+
```

---

# 4. Models / JPA Entities

## Realm

Logical security domain.

**Fields**

- UUID `id`
- String `name`
- boolean `enabled`
- Instant `createdAt`

---

## Client

OAuth2 client application registered under a specific realm.

**Fields**

- UUID `id`
- String `clientId`
- String `clientSecretHash`
- `Set<String>` `grantTypes`
- `Set<String>` `scopes`
- `Realm realm`

---

## ClientRedirectUri

Allowed OAuth2 redirect URIs for a client application.

**Fields**

- UUID `id`
- String `uri`
- `Client client`

---

## User

End-user identity scoped to a realm.

**Fields**

- UUID `id`
- String `username`
- String `email`
- String `passwordHash`
- boolean `enabled`
- `Realm realm`
- `Set<Role>` `roles`

---

## Role

Permissions container associated with users.

**Fields**

- UUID `id`
- String `name`
- String `description`
- `Realm realm`
- `Set<User>` `users`

---

# 5. Repositories (Spring Data JPA)

## RealmRepository

Query realms by unique name.

```java
Optional<Realm> findByName(String name);
```

---

## ClientRepository

Query client applications within a target realm.

```java
Optional<Client> findByClientIdAndRealmName(String clientId, String realmName);
```

---

## UserRepository

Fetch users by username/email within a specific realm.

```java
Optional<User> findByUsernameAndRealmName(String username, String realmName);

Optional<User> findByEmailAndRealmName(String email, String realmName);
```

---

## RoleRepository

Manage role definitions per realm.

```java
Optional<Role> findByNameAndRealmName(String name, String realmName);
```

---

# 6. Configuration Layer

## SecurityConfig

Defines the primary `SecurityFilterChain` for:

- OAuth2 endpoints
- Login pages
- Admin REST APIs

---

## AuthorizationServerConfig

Registers Spring Authorization Server beans.

Responsibilities:

- OAuth2 Authorization Server configuration
- Custom JWT claims
- Token signing
- OIDC support

---

## JwtConfig

Generates and manages the RSA `KeyPair` used to sign JWT Access Tokens and ID Tokens.

---

## PasswordEncoderConfig

Provides:

```java
BCryptPasswordEncoder(12)
```

---

# 7. Controller Endpoints

## Protocol & Discovery Endpoints

| Method | Endpoint | Description |
|---------|----------|-------------|
| GET | `/auth/realms/{realm}/.well-known/openid-configuration` | Returns OIDC Discovery Metadata |
| GET | `/auth/realms/{realm}/.well-known/jwks.json` | Returns public RSA keys |
| GET | `/auth/realms/{realm}/protocol/openid-connect/auth` | Authorization Endpoint (PKCE) |
| POST | `/auth/realms/{realm}/protocol/openid-connect/token` | Exchanges Authorization Code for Tokens |

---

## User Interface & Authentication

| Method | Endpoint | Description |
|---------|----------|-------------|
| GET | `/auth/realms/{realm}/login` | Login Page |
| POST | `/auth/realms/{realm}/login` | Authenticates User |
| GET | `/auth/realms/{realm}/consent` | OAuth2 Consent Page |
| POST | `/auth/realms/{realm}/consent` | Accepts or Rejects Consent |

---

## Administrative REST API

Base Path

```text
/api/v1/admin
```

| Method | Endpoint | Description |
|---------|----------|-------------|
| POST | `/api/v1/admin/realms` | Create Realm |
| GET | `/api/v1/admin/realms` | List Realms |
| POST | `/api/v1/admin/{realm}/clients` | Register OAuth Client |
| GET | `/api/v1/admin/{realm}/clients` | List Clients |
| POST | `/api/v1/admin/{realm}/users` | Create User |
| POST | `/api/v1/admin/{realm}/roles` | Create Role |

---

# 8. Docker Deployment

## Dockerfile

```dockerfile
FROM eclipse-temurin:17-jdk-alpine AS build

WORKDIR /app

COPY . .

RUN ./mvnw clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY --from=build /app/target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

## docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: clavis-db
    environment:
      POSTGRES_DB: clavis_iam
      POSTGRES_USER: clavis
      POSTGRES_PASSWORD: clavis_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  clavis-iam:
    build: .
    container_name: clavis-app
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/clavis_iam
      SPRING_DATASOURCE_USERNAME: clavis
      SPRING_DATASOURCE_PASSWORD: clavis_password
      SPRING_JPA_HIBERNATE_DDL_AUTO: update
    depends_on:
      - postgres

volumes:
  postgres_data:
```

---

## Run

```bash
docker-compose up --build -d
```
