# NexusDesk --- Backend Architecture & Implementation Blueprint

**Status:** Planning baseline. This document is the shared source of
truth for the developer and any coding agent (Claude Code, Codex, etc.).

## 1. Product overview

NexusDesk is an AI-powered enterprise support and knowledge platform. A
company creates a workspace, invites its team, manages support tickets,
uploads internal documents, and uses an AI assistant to ask questions
grounded in those documents. The assistant returns answers with
citations and maintains conversation history.

Target capabilities: 1. Authentication and user identity. 2.
Multi-tenant workspaces, members, roles, teams, and invitations. 3.
Tickets, assignment, comments, status history, and escalation. 4.
Knowledge document upload, extraction, chunking, embeddings, and
indexing. 5. RAG assistant, citations, and conversation history. 6.
Notifications and analytics.

## 2. Locked technology decisions

  -----------------------------------------------------------------------
  Area                                Decision
  ----------------------------------- -----------------------------------
  Language                            Java 21

  Framework                           Spring Boot (choose a compatible
                                      stable release when generating
                                      project)

  Identity                            Keycloak, OpenID Connect

  API security                        Spring Security OAuth2 Resource
                                      Server, JWT

  Database                            PostgreSQL, using the developer's
                                      existing local installation

  Persistence                         Spring Data JPA / Hibernate

  Migrations                          Flyway

  API                                 REST, `/api/v1`

  Build                               Maven

  API docs                            Springdoc OpenAPI

  Tests                               JUnit 5, Mockito, Spring Boot Test,
                                      PostgreSQL Testcontainers where
                                      practical

  Frontend                            React + JavaScript, only after
                                      backend milestones

  Later infrastructure                Kafka, MinIO, vector search,
                                      optionally Redis
  -----------------------------------------------------------------------

**Database ownership rule:** Each microservice owns its database and
schema. Never read or write another service's tables directly.
Communicate through APIs or events.

Use separate databases on the existing PostgreSQL server: `keycloak_db`,
`workspace_db`, and later `ticket_db`, `knowledge_db`, `assistant_db`,
`notification_db`. Keycloak must not use the application Workspace
database.

## 3. Target high-level architecture

``` mermaid
flowchart TB
    UI[React Frontend - later]
    KC[Keycloak Identity Provider]
    GW[API Gateway]
    WS[Workspace Service]
    TS[Ticket Service]
    KS[Knowledge Service]
    AI[AI Assistant Service]
    NS[Notification Service]
    K[(Kafka - later)]
    WDB[(workspace_db)]
    TDB[(ticket_db)]
    KDB[(knowledge_db)]
    ADB[(assistant_db)]
    NDB[(notification_db)]
    OBJ[(MinIO - later)]
    LLM[LLM Provider]

    UI -->|OIDC login| KC
    UI -->|Bearer JWT| GW
    GW --> WS
    GW --> TS
    GW --> KS
    GW --> AI
    GW --> NS
    WS --> WDB
    TS --> TDB
    KS --> KDB
    KS --> OBJ
    AI --> ADB
    AI -->|authorized retrieval| KS
    AI --> LLM
    WS -. events .-> K
    TS -. events .-> K
    KS -. events .-> K
    K -. consume .-> NS
    NS --> NDB
```

### Services and ownership

  -----------------------------------------------------------------------
  Service                 Responsibility          Database
  ----------------------- ----------------------- -----------------------
  Workspace Service       Workspaces, members,    `workspace_db`
                          roles, teams,           
                          invitations             

  Ticket Service          Tickets, comments,      `ticket_db`
                          assignments, history,   
                          escalation              

  Knowledge Service       Documents, ingestion    `knowledge_db`
                          jobs, chunk metadata,   
                          vector indexing         

  AI Assistant Service    Conversations,          `assistant_db`
                          messages, feedback, RAG 
                          orchestration           

  Notification Service    Notifications and       `notification_db`
                          event-consumer state    

  API Gateway             Routing and edge        None
                          concerns; no business   
                          database                

  Keycloak                Identity, credentials,  `keycloak_db`
                          login, token issuance   
  -----------------------------------------------------------------------

### Implementation order

1.  Keycloak local setup.
2.  Workspace Service, fully tested.
3.  Ticket Service.
4.  API Gateway and service routing/security integration.
5.  Knowledge Service.
6.  AI Assistant and RAG.
7.  Kafka and Notification Service.
8.  Outbox and Saga where justified.
9.  Integration tests, observability, deployment hardening.
10. React frontend only after backend milestones are stable.

Do not let Eureka/service discovery delay the first working version; use
explicit local service URLs initially. Gateway can be scaffolded earlier
if useful, but first prove the Workspace Service independently.

## 4. Keycloak --- setup now

### Local values

  ---------------------------------------------------------------------------------------------------------------
  Setting                             Value
  ----------------------------------- ---------------------------------------------------------------------------
  Keycloak                            `http://localhost:8080`

  Admin console                       `http://localhost:8080/admin`

  Realm                               `nexusdesk`

  Realm issuer                        `http://localhost:8080/realms/nexusdesk`

  OIDC discovery                      `http://localhost:8080/realms/nexusdesk/.well-known/openid-configuration`

  Workspace Service                   `http://localhost:8081`

  Workspace API                       `http://localhost:8081/api/v1`

  Future React URL                    `http://localhost:5173`
  ---------------------------------------------------------------------------------------------------------------

### Run Keycloak with the existing PostgreSQL server

Create a dedicated database `keycloak_db` and dedicated PostgreSQL user
`keycloak_user`. Do not use the PostgreSQL superuser for the Keycloak
runtime.

Example `infrastructure/docker-compose.yml` service (adjust credentials
to your local setup):

``` yaml
services:
  keycloak:
    image: quay.io/keycloak/keycloak:26.8.2
    container_name: nexusdesk-keycloak
    command: start-dev
    environment:
      KC_BOOTSTRAP_ADMIN_USERNAME: ${KEYCLOAK_ADMIN}
      KC_BOOTSTRAP_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://host.docker.internal:5432/keycloak_db
      KC_DB_USERNAME: ${KEYCLOAK_DB_USERNAME}
      KC_DB_PASSWORD: ${KEYCLOAK_DB_PASSWORD}
    ports:
      - "127.0.0.1:8080:8080"
    restart: unless-stopped
```

Example ignored local `.env` (never commit this file):

``` dotenv
KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=change-this-local-password
KEYCLOAK_DB_USERNAME=keycloak_user
KEYCLOAK_DB_PASSWORD=change-this-db-password
```

`host.docker.internal` is commonly available from Docker Desktop on
Windows for containers to reach host services. PostgreSQL must accept
the connection from Docker Desktop; if it fails, inspect PostgreSQL
listen address, `pg_hba.conf`, firewall, host and port. Bind Keycloak to
localhost for local development. `start-dev` is development-only, not
production. Pin the image version rather than `latest`.

### Create realm and clients

1.  Open `http://localhost:8080/admin`.
2.  Create realm `nexusdesk`; do not use `master` for application users.
3.  Create a future browser client `nexusdesk-web`:
    -   Client type: OpenID Connect.
    -   Client authentication: OFF (public client).
    -   Standard flow: ON.
    -   Direct access grants: OFF.
    -   Implicit flow: OFF.
    -   Use Authorization Code + PKCE with `S256`.
    -   Valid redirect URI: `http://localhost:5173/*`.
    -   Valid post-logout redirect URI: `http://localhost:5173/*`.
    -   Web origin: `http://localhost:5173`.
4.  Create a separate API audience/resource identifier `nexusdesk-api`.
    Configure an audience mapper so access tokens intended for this API
    contain `nexusdesk-api` in `aud`. Do not enable browser login flow
    for this API identifier.
5.  Create a test user in the `nexusdesk` realm and set local
    development credentials.

**Why redirect URLs?** When a browser application sends a user to
Keycloak for login, Keycloak must know which URL is allowed to receive
the browser after login. `Valid redirect URIs` is an allow-list.
`Web origins` configures allowed browser origins for cross-origin
requests. Use exact URLs and narrow patterns; never use wildcard
redirect URIs or wildcard web origins in production.

The frontend client is configured ahead of time, but frontend
development remains postponed. Do not enable Resource Owner Password
Credentials / Direct Access Grants for the application login flow. Do
not put client secrets in React.

### Keycloak vs Workspace Service responsibilities

  Responsibility                                         Owner
  ------------------------------------------------------ ---------------------------
  Passwords, credentials, login/logout, token issuance   Keycloak
  External identity ID (`sub`)                           Keycloak
  User email/name profile                                Keycloak identity profile
  Workspace creation and lifecycle                       Workspace Service
  Workspace membership and invitations                   Workspace Service
  Workspace-specific OWNER/ADMIN/AGENT/MEMBER role       Workspace Service
  Team membership                                        Workspace Service
  Authorization to access a workspace                    Workspace Service

A person can be ADMIN in Workspace A and MEMBER in Workspace B.
Therefore workspace roles must be stored in our database, not as global
Keycloak roles. Use the verified JWT `sub` as the external identity key,
not email.

## 5. Workspace Service domain model

The Workspace Service is the first service to implement. It owns
workspace lifecycle, membership, workspace-scoped roles, teams, and
invitations. It does not manage passwords or perform custom login.

### Entities

#### Workspace

  Field                        Type / rule
  ---------------------------- -----------------------------------
  `id`                         UUID primary key
  `name`                       varchar(120), required
  `slug`                       varchar(80), unique, URL-friendly
  `description`                varchar(500), optional
  `status`                     ACTIVE, SUSPENDED, ARCHIVED
  `created_by`                 Keycloak `sub`
  `created_at`, `updated_at`   timestamptz

#### WorkspaceMember

  Field                                     Type / rule
  ----------------------------------------- -------------------------------------
  `id`                                      UUID primary key
  `workspace_id`                            FK to Workspace
  `keycloak_user_id`                        JWT `sub`, varchar(100)
  `email`                                   varchar(255), profile snapshot
  `display_name`                            varchar(150), optional snapshot
  `role`                                    OWNER, ADMIN, AGENT, MEMBER
  `status`                                  ACTIVE, INVITED, SUSPENDED, REMOVED
  `joined_at`, `created_at`, `updated_at`   timestamptz

Constraints: unique `(workspace_id, keycloak_user_id)`; index
`(keycloak_user_id, status)` and `(workspace_id, role, status)`. Do not
use email as the identity key.

#### Team

  Field                        Type / rule
  ---------------------------- ------------------------
  `id`                         UUID primary key
  `workspace_id`               FK to Workspace
  `name`                       varchar(100), required
  `description`                varchar(500), optional
  `created_at`, `updated_at`   timestamptz

Constraint: unique `(workspace_id, name)`; consider case-insensitive
uniqueness.

#### TeamMember

  Field                   Type / rule
  ----------------------- -----------------------
  `id`                    UUID primary key
  `team_id`               FK to Team
  `workspace_member_id`   FK to WorkspaceMember
  `created_at`            timestamptz

Constraint: unique `(team_id, workspace_member_id)`. A team member must
be an active member of the same workspace as the team.

#### WorkspaceInvitation

  Field                            Type / rule
  -------------------------------- ----------------------------------------------
  `id`                             UUID primary key
  `workspace_id`                   FK to Workspace
  `email`                          normalized varchar(255)
  `role`                           ADMIN, AGENT, MEMBER; not OWNER by default
  `token_hash`                     hash of high-entropy random invitation token
  `status`                         PENDING, ACCEPTED, EXPIRED, REVOKED
  `invited_by`                     Keycloak `sub`
  `expires_at`                     timestamptz
  `accepted_by_keycloak_user_id`   nullable varchar(100)
  `accepted_at`, `created_at`      timestamptz

Never store raw invitation tokens. Tokens must be single-use and expire.
Acceptance must verify the authenticated user's email against the
invitation email according to the verified-email policy.

Optional later entity: `WorkspaceAuditLog` for actor, workspace, action,
resource type/id, timestamp, and safe metadata. Do not store secrets or
raw tokens.

### Entity relationship diagram

``` mermaid
erDiagram
    WORKSPACE ||--o{ WORKSPACE_MEMBER : has
    WORKSPACE ||--o{ TEAM : contains
    WORKSPACE ||--o{ WORKSPACE_INVITATION : issues
    TEAM ||--o{ TEAM_MEMBER : has
    WORKSPACE_MEMBER ||--o{ TEAM_MEMBER : joins
    WORKSPACE {
      uuid id PK
      varchar name
      varchar slug UK
      varchar status
      varchar created_by
    }
    WORKSPACE_MEMBER {
      uuid id PK
      uuid workspace_id FK
      varchar keycloak_user_id
      varchar role
      varchar status
    }
    TEAM {
      uuid id PK
      uuid workspace_id FK
      varchar name
    }
    TEAM_MEMBER {
      uuid id PK
      uuid team_id FK
      uuid workspace_member_id FK
    }
    WORKSPACE_INVITATION {
      uuid id PK
      uuid workspace_id FK
      varchar email
      varchar role
      varchar token_hash
      varchar status
      timestamptz expires_at
    }
```

### Initial role permissions

  Action                        OWNER                        ADMIN   AGENT   MEMBER
  --------------------------- ------- ---------------------------- ------- --------
  View workspace                  Yes                          Yes     Yes      Yes
  Update workspace settings       Yes                          Yes      No       No
  Invite members                  Yes                          Yes      No       No
  Change member roles             Yes   Yes, except OWNER transfer      No       No
  Remove members                  Yes   Yes, with OWNER safeguards      No       No
  Create/manage teams             Yes                          Yes      No       No
  View teams                      Yes                          Yes     Yes      Yes

Only OWNER can transfer ownership. ADMIN cannot grant OWNER. Never allow
removal/demotion of the last active OWNER. Ticket-specific permissions
will be defined in Ticket Service.

## 6. Workspace Service REST endpoints

Base path: `/api/v1`. All business endpoints require a valid access
token. Every workspace-scoped operation checks membership and role in
the Workspace Service database.

### Workspace

  ----------------------------------------------------------------------------------------
  Method                  Endpoint                                 Purpose
  ----------------------- ---------------------------------------- -----------------------
  POST                    `/workspaces`                            Create workspace;
                                                                   caller becomes OWNER

  GET                     `/workspaces`                            List current user's
                                                                   workspaces

  GET                     `/workspaces/{workspaceId}`              Get workspace if member

  PATCH                   `/workspaces/{workspaceId}`              Update workspace

  GET                     `/workspaces/{workspaceId}/members/me`   Current user's
                                                                   membership/role

  POST                    `/workspaces/{workspaceId}/archive`      Archive workspace
                                                                   (OWNER)
  ----------------------------------------------------------------------------------------

### Members and invitations

  -------------------------------------------------------------------------------------------------------------------
  Method                  Endpoint                                                        Purpose
  ----------------------- --------------------------------------------------------------- ---------------------------
  GET                     `/workspaces/{workspaceId}/members`                             List members

  GET                     `/workspaces/{workspaceId}/members/{memberId}`                  Get member

  PATCH                   `/workspaces/{workspaceId}/members/{memberId}/role`             Change role with safeguards

  PATCH                   `/workspaces/{workspaceId}/members/{memberId}/status`           Suspend/reactivate/remove

  POST                    `/workspaces/{workspaceId}/invitations`                         Create invitation

  GET                     `/workspaces/{workspaceId}/invitations`                         List invitations

  POST                    `/invitations/accept`                                           Accept invitation using
                                                                                          authenticated identity and
                                                                                          token

  POST                    `/workspaces/{workspaceId}/invitations/{invitationId}/resend`   Resend pending invitation

  DELETE                  `/workspaces/{workspaceId}/invitations/{invitationId}`          Revoke invitation
  -------------------------------------------------------------------------------------------------------------------

### Teams

  ---------------------------------------------------------------------------------------------------------------
  Method                  Endpoint                                                        Purpose
  ----------------------- --------------------------------------------------------------- -----------------------
  POST                    `/workspaces/{workspaceId}/teams`                               Create team

  GET                     `/workspaces/{workspaceId}/teams`                               List teams

  GET                     `/workspaces/{workspaceId}/teams/{teamId}`                      Team details

  PATCH                   `/workspaces/{workspaceId}/teams/{teamId}`                      Update team

  DELETE                  `/workspaces/{workspaceId}/teams/{teamId}`                      Delete/archive team

  POST                    `/workspaces/{workspaceId}/teams/{teamId}/members`              Add active workspace
                                                                                          member

  GET                     `/workspaces/{workspaceId}/teams/{teamId}/members`              List team members

  DELETE                  `/workspaces/{workspaceId}/teams/{teamId}/members/{memberId}`   Remove team member
  ---------------------------------------------------------------------------------------------------------------

### Current user

  Method   Endpoint   Purpose
  -------- ---------- -------------------------------------------------------
  GET      `/me`      Return safe identity claims and workspace memberships

### API conventions

-   Use DTOs; never expose JPA entities directly.
-   Use Jakarta Bean Validation.
-   Use consistent `ProblemDetail` or documented error DTO.
-   Use `201 Created`, `200 OK`, and `204 No Content` appropriately.
-   Paginate potentially large lists.
-   Use UUID path parameters and request/correlation IDs in logs.
-   Derive `createdBy`, user identity, and caller role from verified
    security context, never request JSON.
-   Use transactions for workspace creation + owner membership.
-   Use optimistic locking (`@Version`) where concurrent updates matter.

Example create request:

``` http
POST /api/v1/workspaces
Authorization: Bearer <access-token>
Content-Type: application/json
```

``` json
{
  "name": "Acme Support",
  "slug": "acme-support",
  "description": "Internal support workspace"
}
```

The service reads the caller's Keycloak `sub`, creates Workspace and
WorkspaceMember with role OWNER/status ACTIVE in one transaction.

## 7. Workspace Service project structure

``` text
workspace-service/
├── src/main/java/com/nexusdesk/workspace/
│   ├── WorkspaceServiceApplication.java
│   ├── config/              # SecurityConfig, OpenApiConfig
│   ├── common/              # exception, response, audit
│   ├── security/            # CurrentUser, JwtPrincipalService
│   ├── workspace/           # controller, service, repository, entity, dto
│   ├── member/              # controller, service, repository, entity, dto
│   ├── team/                # controller, service, repository, entity, dto
│   └── invitation/          # controller, service, repository, entity, dto
├── src/main/resources/
│   ├── application.yml
│   └── db/migration/
│       ├── V1__create_workspace_tables.sql
│       └── V2__add_workspace_indexes.sql
├── src/test/java/
└── pom.xml
```

Use feature-oriented packages. Initially each service is an independent
Maven project; do not prematurely create a large shared multi-module
build or share service-owned domain entities.

### Dependencies

-   Spring Web
-   Spring Data JPA
-   PostgreSQL Driver
-   Spring Security
-   OAuth2 Resource Server
-   Validation
-   Flyway core and PostgreSQL module compatible with chosen Flyway
    release
-   Spring Boot Actuator
-   Springdoc OpenAPI
-   Lombok (optional)
-   Spring Boot Test
-   Spring Security Test
-   Testcontainers PostgreSQL (recommended for integration tests)

Use Spring Boot dependency management. Do not manually mix incompatible
Spring Framework/Security versions.

### Example `application.yml`

``` yaml
server:
  port: 8081

spring:
  application:
    name: workspace-service
  datasource:
    url: ${WORKSPACE_DB_URL:jdbc:postgresql://localhost:5432/workspace_db}
    username: ${WORKSPACE_DB_USERNAME:postgres}
    password: ${WORKSPACE_DB_PASSWORD}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        format_sql: true
  flyway:
    enabled: true
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${KEYCLOAK_ISSUER_URI:http://localhost:8080/realms/nexusdesk}

management:
  endpoints:
    web:
      exposure:
        include: health,info
```

Use `ddl-auto: validate`; Flyway owns schema changes. Configure issuer
to exactly match token `iss`. Validate audience after confirming the
Keycloak mapper's actual `aud` claim. Never commit real secrets. If the
service runs inside Docker, `localhost` refers to that container, so use
correct host/service DNS.

## 8. Security and tenant isolation

### Authentication flow

1.  Client authenticates with Keycloak using OIDC Authorization Code +
    PKCE.
2.  Client sends `Authorization: Bearer <token>`.
3.  Spring Security validates signature via Keycloak JWK Set and
    validates issuer/time claims.
4.  Validate intended API audience.
5.  Extract verified `sub`.
6.  Check active workspace membership and role in PostgreSQL.
7.  Execute the operation only if authorized.

Spring Security's OAuth2 Resource Server supports JWT validation through
issuer discovery/JWKs. `issuer-uri` must match the token's `iss`.
Audience validation is also required for this API.

### Rules

-   Health endpoints may be public; business APIs require
    authentication.
-   Check membership on every workspace-scoped operation.
-   A workspace ID in the URL is not proof of access.
-   Never trust user ID, role, or `createdBy` from request bodies.
-   Prevent IDOR by checking resource ownership/tenant on every read and
    write.
-   Do not log access/refresh tokens or invitation secrets.
-   Configure narrow CORS origins when the frontend is introduced.
-   Production requires HTTPS, production Keycloak mode, secure secret
    management, and key/credential rotation.

Suggested application authorization methods: -
`requireActiveMembership(workspaceId, keycloakUserId)` -
`requireRole(workspaceId, keycloakUserId, allowedRoles)` -
`requireOwner(workspaceId, keycloakUserId)`

Do not duplicate authorization logic in controllers. Enforce business
authorization in the service layer. Method security may supplement, not
replace, workspace-scoped checks.

### Claims to inspect in a development token

-   `iss`: exact realm issuer.
-   `sub`: stable Keycloak identity.
-   `exp`, `iat`: token timestamps.
-   `aud`: must contain API audience.
-   `preferred_username`, `email`, `name`: optional profile claims.

Use `sub` as identity key; email is mutable.

## 9. Database rules

-   `keycloak_db` is for Keycloak only.
-   `workspace_db` is for Workspace Service only.
-   UUID primary keys.
-   `timestamptz` timestamps.
-   Unique constraints for slug and membership/team membership.
-   Index common membership and tenant-scoped access paths.
-   Foreign keys only within the owning service database.
-   No foreign key to Keycloak internal tables.
-   Workspace + owner membership created in one local transaction.
-   Use status/archival for lifecycle where history must be retained.
-   Keep Flyway scripts as schema source of truth.

Initial Workspace Service tables: `workspaces`, `workspace_members`,
`teams`, `team_members`, `workspace_invitations`.

## 10. Tests and acceptance criteria

### Keycloak

-   Keycloak starts and retains state in dedicated PostgreSQL database.
-   `nexusdesk` realm, clients, and test user exist.
-   Test user authenticates.
-   Token `iss` matches configured issuer.
-   Token has expected `sub` and API audience.
-   Expired, malformed, wrong-issuer, and wrong-audience tokens are
    rejected.

### Workspace Service

-   No token -\> 401 on protected endpoints.
-   Invalid/expired JWT -\> 401.
-   Authenticated user can create a workspace.
-   Creator becomes OWNER/ACTIVE atomically.
-   User lists only their own workspaces.
-   User A cannot read/update User B's workspace without membership.
-   ADMIN cannot grant OWNER or remove last OWNER.
-   Duplicate slug -\> conflict.
-   Duplicate membership prevented by DB constraint.
-   Invitation token hashed, single-use, expires.
-   Team cannot add non-member or member from another workspace.
-   Migrations work from an empty database.
-   Validation and error responses are consistent.

**First end-to-end milestone:** Authenticate a test user with Keycloak,
call Workspace Service using a bearer token, create a workspace, then
retrieve it from that user's workspace list. Use Postman/curl; React is
not needed.

## 11. Later service plans --- do not implement yet

### Ticket Service

Entities: `Ticket`, `TicketComment`, `TicketHistory`, later
`TicketAttachment` and possibly `TicketEscalation`.

Initial API direction: - `POST /api/v1/tickets` -
`GET /api/v1/tickets` - `GET /api/v1/tickets/{ticketId}` -
`PATCH /api/v1/tickets/{ticketId}/status` -
`PATCH /api/v1/tickets/{ticketId}/assignee` -
`POST /api/v1/tickets/{ticketId}/comments` -
`GET /api/v1/tickets/{ticketId}/history` -
`POST /api/v1/tickets/{ticketId}/escalations`

Every ticket belongs to a workspace. Ticket Service independently
verifies authorization through a deliberate service-to-service strategy;
it must not query Workspace Service's DB directly.

### Knowledge Service

Entities: `KnowledgeDocument`, `DocumentIngestionJob`, optionally
relational `DocumentChunk` metadata and `DocumentAccessPolicy`.

API direction: - `POST /api/v1/knowledge/documents` -
`GET /api/v1/knowledge/documents` -
`GET /api/v1/knowledge/documents/{documentId}` -
`POST /api/v1/knowledge/documents/{documentId}/retry` -
`DELETE /api/v1/knowledge/documents/{documentId}` - Internal authorized
retrieval endpoint.

Ingestion: authorize workspace -\> validate file -\> store original
(MinIO later) -\> persist document/job -\> extract text -\> chunk -\>
embed -\> persist vector + metadata -\> mark INDEXED. On failure, record
safe failure details and retry state. Metadata should include
workspaceId, documentId, title, pageNumber, chunkIndex. Enforce tenant
filtering during retrieval; never rely on prompt instructions to enforce
data isolation.

### AI Assistant Service

Entities: `Conversation`, `ChatMessage`, `MessageSource`,
`MessageFeedback`, later `ConversationSummary`.

API direction: - `POST /api/v1/assistant/conversations` -
`GET /api/v1/assistant/conversations` -
`GET /api/v1/assistant/conversations/{conversationId}` -
`POST /api/v1/assistant/conversations/{conversationId}/messages` -
`POST /api/v1/assistant/messages/{messageId}/feedback`

RAG flow: validate identity/workspace -\> load bounded chat history -\>
embed query -\> retrieve workspace-filtered chunks -\> construct prompt
-\> Spring AI `ChatClient` -\> return answer and citations -\> persist
messages and source references.

Conversation history and RAG are different: history is what the
user/assistant said; RAG retrieves relevant knowledge-base facts. Start
with bounded recent history; add summarization later.

### Notification Service

Entities: `Notification`, later `NotificationPreference`,
`ProcessedEvent`.

API direction: - `GET /api/v1/notifications` -
`PATCH /api/v1/notifications/{notificationId}/read` -
`PATCH /api/v1/notifications/read-all`

Event examples: `WorkspaceCreated`, `MemberInvited`, `TicketCreated`,
`TicketAssigned`, `TicketResolved`, `DocumentUploaded`,
`DocumentIndexed`, `DocumentProcessingFailed`.

Event envelope: `eventId`, `eventType`, `eventVersion`, `occurredAt`,
`workspaceId`, `aggregateId`, `producer`, `payload`. Consumers must be
idempotent; unique `(consumer_name, event_id)` in `processed_event` is
one option.

## 12. Outbox and Saga --- later

**Outbox:** update business data and insert an outbox row in the same
local transaction; publisher sends pending rows to Kafka; mark published
after broker acknowledgement; retry failures with backoff. Outbox
preserves publish intent but does not guarantee exactly-once delivery.
Consumers still need idempotency.

**Saga:** only introduce for a real multi-service workflow. Possible
workspace onboarding: create workspace -\> provision knowledge area -\>
create default team -\> activate workspace -\> welcome notification.
Track workflow state and use compensating business actions, not
distributed database rollback.

## 13. Repository structure

``` text
nexusdesk/
├── infrastructure/
│   ├── docker-compose.yml
│   ├── keycloak/
│   └── database/
├── backend/
│   ├── workspace-service/
│   ├── ticket-service/
│   ├── api-gateway/
│   ├── knowledge-service/
│   ├── ai-assistant-service/
│   └── notification-service/
├── frontend/                 # create after backend milestones
└── docs/
    └── NEXUSDESK_BACKEND_BLUEPRINT.md
```

Each service starts as an independent Maven project. Avoid premature
shared domain libraries.

## 14. Phased implementation checklist

### Phase 0 --- local foundation

-   [ ] Create repository/folder structure.
-   [ ] Confirm PostgreSQL version, port, user, and connectivity.
-   [ ] Create dedicated databases/users for Keycloak and Workspace
    Service.
-   [ ] Add `.gitignore` for `.env`, secrets, build output, IDE files.
-   [ ] Add Keycloak Compose configuration.
-   [ ] Start Keycloak and verify admin console.

### Phase 1 --- Keycloak

-   [ ] Create `nexusdesk` realm.
-   [ ] Create `nexusdesk-web` public client with Authorization Code +
    PKCE S256 and future React redirect URLs.
-   [ ] Create `nexusdesk-api` audience/resource identifier and mapper.
-   [ ] Create test user.
-   [ ] Obtain a test access token.
-   [ ] Verify `iss`, `sub`, `exp`, `aud`.
-   [ ] Keep secrets out of Git.

### Phase 2 --- Workspace Service foundation

-   [ ] Generate Maven Spring Boot project.
-   [ ] Configure PostgreSQL.
-   [ ] Configure Flyway and first migration.
-   [ ] Configure OAuth2 Resource Server.
-   [ ] Validate issuer and audience.
-   [ ] Add health endpoint and OpenAPI.
-   [ ] Verify 401 without token and successful authenticated test
    endpoint.

### Phase 3 --- workspace core

-   [ ] Implement Workspace and WorkspaceMember.
-   [ ] Implement repositories, DTOs, and validation.
-   [ ] Implement transactional create workspace + OWNER membership.
-   [ ] Implement list current user's workspaces.
-   [ ] Implement workspace detail/update.
-   [ ] Add authorization and tenant isolation tests.

### Phase 4 --- members and invitations

-   [ ] Implement membership lifecycle and role changes.
-   [ ] Implement invitation token hashing, expiry, revoke, accept.
-   [ ] Enforce owner/admin safeguards.
-   [ ] Test privilege escalation and invitation edge cases.

### Phase 5 --- teams

-   [ ] Implement Team and TeamMember.
-   [ ] Implement create/list/update/archive APIs.
-   [ ] Implement add/remove active members.
-   [ ] Test cross-workspace membership rejection.

### Phase 6 --- finish Workspace Service

-   [ ] Standardize errors and validation.
-   [ ] Add pagination/sorting.
-   [ ] Add PostgreSQL integration tests.
-   [ ] Add OpenAPI examples and service README.
-   [ ] Verify all acceptance criteria.

Then proceed to Ticket, Gateway, Knowledge, AI, Kafka/Notifications,
Outbox/Saga, deployment, and finally React.

## 15. Suggested first-week schedule

  Day   Deliverable
  ----- ------------------------------------------------------------
  1     Confirm PostgreSQL; create databases/users; start Keycloak
  2     Create realm/clients/test user; inspect JWT
  3     Generate Workspace Service; configure DB, Flyway, security
  4     Implement Workspace and WorkspaceMember; create API
  5     Implement list/get/update and tenant isolation
  6     Implement invitations and role safeguards
  7     Implement teams, integration tests, API docs

This is a guide, not a reason to skip tests or security checks.

## 16. Coding-agent instructions

When this file is supplied to Claude Code, Codex, or another coding
agent:

1.  Treat this file as the project's source of truth.
2.  Implement only the phase explicitly requested. Do not generate all
    services at once.
3.  Inspect the existing repository before editing; report structure and
    relevant configuration.
4.  Do not overwrite existing files without explaining the impact.
5.  Use Java 21 and a compatible Spring Boot release; use Spring Boot
    dependency management.
6.  Use PostgreSQL, Spring Data JPA, and Flyway. Do not substitute
    MariaDB/MySQL.
7.  Use Keycloak + Spring Security OAuth2 Resource Server. Do not
    implement custom password authentication.
8.  Use verified JWT `sub` as external user identity.
9.  Keep workspace-specific roles and membership in Workspace Service.
10. Enforce tenant isolation on every workspace-scoped operation.
11. Use DTOs, validation, service-layer rules, transactions, and
    consistent errors.
12. Test both allowed and forbidden/cross-tenant cases.
13. Do not add Kafka, Redis, MinIO, Kubernetes, Eureka, vector search,
    or Spring AI before their planned phases.
14. Do not create React until backend milestones are complete.
15. After each step, report files changed, commands, tests, results, and
    manual configuration still required.
16. If requirements conflict or are ambiguous, ask before changing
    architecture.

### First coding-agent task

> Read `NEXUSDESK_BACKEND_BLUEPRINT.md` completely. We are starting
> Phase 0 and Phase 1 only. First inspect my existing repository and
> PostgreSQL setup assumptions. Do not implement Workspace Service yet.
> Prepare local Keycloak + PostgreSQL configuration using a dedicated
> `keycloak_db`, explain host-specific values I must fill in, create
> safe `.env.example` and `.gitignore` if missing, and provide exact
> commands to start and verify Keycloak. Do not commit secrets. Do not
> add frontend or other backend services.

## 17. Official references

-   Keycloak Docker getting started:
    https://www.keycloak.org/getting-started/getting-started-docker
-   Keycloak container configuration:
    https://www.keycloak.org/server/containers
-   Spring Security OAuth2 Resource Server JWT:
    https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/jwt.html
-   Spring Boot OAuth2 configuration:
    https://docs.spring.io/spring-boot/reference/security/oauth2.html

Confirm exact dependency compatibility against the Spring Boot version
selected during project generation.

## 18. Current checkpoint

**Current phase:** Planning complete; implementation not yet started.

**First action:** Confirm local PostgreSQL connection details and create
dedicated `keycloak_db` database/user. Then start Keycloak, create the
realm and clients, and verify a real JWT.

**Definition of done for the first checkpoint:** Keycloak runs locally
with persistent PostgreSQL storage, `nexusdesk` realm exists, a test
user authenticates, and the access token has the expected issuer and
subject. Configure and verify the API audience before the Workspace
Service enforces it.

Do not start Ticket Service, Kafka, RAG, React, or the other services
yet.
