# Clavis-IAM 🔑

Clavis-IAM is a lightweight, monolithic Identity and Access Management (IAM) server built with **Spring Boot 3**, **Spring Security 6**, and **Spring Data JPA** backed by **PostgreSQL**[cite: 1]. 

Designed as a clean alternative to legacy enterprise tools like Keycloak, it delivers a production-ready OAuth 2.0 and OpenID Connect (OIDC) engine optimized for modern web applications and microservices[cite: 1].

---

# 1. Project Overview

Clavis-IAM implements standard authorization and authentication flows cleanly within a single Spring Boot application

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
| **Persistence** | Spring Data JPA, PostgreSQL 15, Flyway Migrations |
| **Frontend UI** | React, Tailwind CSS |
| **Observability & Docs** | Spring Boot Actuator, SpringDoc OpenAPI |
| **Testing** | JUnit 5, Testcontainers|
| **Containerization** | Docker, Docker Compose, GitHub Actions CI |

---

# 3. Engineering Design Trade-Offs & Decisions

When building Clavis-IAM, several architectural decisions were made to balance implementation velocity, enterprise security standards, and operational simplicity:

1. **Monolithic vs. Microservice IAM:**
   * *Trade-off:* Chose a modular monolith instead of a distributed IAM cluster.
   * *Why:* For small-to-medium systems, a monolith eliminates distributed tracing complexity, network hops, and operational overhead while still maintaining strict logical boundaries via multi-tenant realms.
2. **Stateless JWTs vs. State Sessions:**
   * *Trade-off:* Relied on cryptographically signed JWTs paired with a secure refresh token rotation strategy.
   * *Why:* Offloads token validation overhead from the database to individual resource servers via JWKS distribution, while token rotation mitigates leakage risks.
3. **Flyway vs. Hibernate Auto DDL:**
   * *Trade-off:* Swapped `spring.jpa.hibernate.ddl-auto=update` for explicit version-controlled Flyway migrations.
   * *Why:* Essential for production environments to prevent unintended destructive schema modifications and maintain a predictable audit trail of database evolutions.

---

# 4. Architecture & Domain Model

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
          │                     └── Refresh Tokens (with Rotation)
          │
          ├── Redirect URIs
          ├── Client Secrets
          └── Allowed Scopes
```[cite: 1]

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
```[cite: 1]

---

# 5. Models / JPA Entities

## Realm
Logical security domain.
- UUID `id`, String `name`, boolean `enabled`, Instant `createdAt`

## Client
OAuth2 client application registered under a specific realm.
- UUID `id`, String `clientId`, String `clientSecretHash`, `Set<String>` `grantTypes`, `Set<String>` `scopes`, `Realm realm`

## ClientRedirectUri
Allowed OAuth2 redirect URIs for a client application.
- UUID `id`, String `uri`, `Client client`

## User
End-user identity scoped to a realm.
- UUID `id`, String `username`, String `email`, String `passwordHash`, boolean `enabled`, `Realm realm`, `Set<Role>` `roles`

## Role
Permissions container associated with users.
- UUID `id`, String `name`, String `description`, `Realm realm`, `Set<User>` `users`

---

# 6. Repositories (Spring Data JPA)

- **RealmRepository:** `Optional<Realm> findByName(String name);
- **ClientRepository:** `Optional<Client> findByClientIdAndRealmName(String clientId, String realmName);
- **UserRepository:** `Optional<User> findByUsernameAndRealmName(String username, String realmName);`, `Optional<User> findByEmailAndRealmName(String email, String realmName);
- **RoleRepository:** `Optional<Role> findByNameAndRealmName(String name, String realmName);

---

# 7. Configuration & Security Layer

- **SecurityConfig:** Defines the primary `SecurityFilterChain` for OAuth2 endpoints, login pages, and secured administrative APIs.
- **AuthorizationServerConfig:** Configures Spring Authorization Server beans, token signing, and OIDC support.
- **JwtConfig:** Manages the RSA `KeyPair` for signing JWT Access and ID Tokens.
- **PasswordEncoderConfig:** Provides `BCryptPasswordEncoder(12).

---

# 8. Controller Endpoints

## Protocol & Discovery Endpoints
| Method | Endpoint | Description |
|---|---|---|
| GET | `/auth/realms/{realm}/.well-known/openid-configuration` | OIDC Discovery Metadata |
| GET | `/auth/realms/{realm}/.well-known/jwks.json` | Public RSA keys |
| GET | `/auth/realms/{realm}/protocol/openid-connect/auth` | Authorization Endpoint (PKCE) |
| POST | `/auth/realms/{realm}/protocol/openid-connect/token` | Token Exchange & Refresh Rotation |

## Administrative REST API (`/api/v1/admin`)
Protected via strict admin authentication models and documented via **SpringDoc OpenAPI** (`/swagger-ui.html`).

---

# 9. Docker & Production Deployment

## docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: clavis-db
    environment:
      POSTGRES_DB: ${DB_NAME:-clavis_iam}
      POSTGRES_USER: ${DB_USER:-clavis}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
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
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/${DB_NAME:-clavis_iam}
      SPRING_DATASOURCE_USERNAME: ${DB_USER:-clavis}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      SPRING_FLYWAY_ENABLED: "true"
    depends_on:
      - postgres

volumes:
  postgres_data:
```[cite: 1]

## Running the Application

1. Set up your local environment file (`.env`) containing your production secrets (database credentials and RSA keys).
2. Start the services using Docker Compose[cite: 1]:
   ```bash
   docker-compose up --build -d
