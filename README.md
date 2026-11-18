# NexusDesk

**NexusDesk** is an AI-powered enterprise support and knowledge platform.

It is a **microservices** system built with Spring Boot 4.x + Java 17, a React
frontend, and PostgreSQL. Each service lives in its own repository so every team
can manage its own lifecycle, CI/CD, and versioning independently.

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Spring Boot 4.x, Java 17, Spring Security, Spring Data JPA |
| Cloud | Spring Cloud (Eureka, OpenFeign, API Gateway) |
| Security | Keycloak 26.8.2 — JWT/OAuth2 resource server |
| Messaging | Kafka (inter-service communication) |
| Resilience | Resilience4j circuit breakers |
| Database | PostgreSQL 17 (one database per service) |
| Frontend | React.js, Vite, Tailwind CSS, shadcn/ui |
| Build | Maven |

## Repositories

This repository is the **umbrella** — it holds only shared files. Every service
is its own repository under the `Gnamdev` GitHub account:

| Repository | Service | Purpose |
|---|---|---|
| [`nexusdesk-workspace-service`](https://github.com/Gnamdev/nexusdesk-workspace-service) | Workspace service | Workspaces, members, invitations, teams |
| [`nexusdesk-ticket-service`](https://github.com/Gnamdev/nexusdesk-ticket-service) | Ticket service | Tickets, comments, history, escalations |
| [`nexusdesk-api-gateway`](https://github.com/Gnamdev/nexusdesk-api-gateway) | API gateway | Single entry point, routing, auth |
| [`nexusdesk-eureka-service`](https://github.com/Gnamdev/nexusdesk-eureka-service) | Eureka server | Service discovery |
| [`nexusdesk-knowledge-service`](https://github.com/Gnamdev/nexusdesk-knowledge-service) | Knowledge service | Knowledge base articles |
| [`nexusdesk-ai-assistant-service`](https://github.com/Gnamdev/nexusdesk-ai-assistant-service) | AI assistant service | AI-powered support chat |
| [`nexusdesk-notification-service`](https://github.com/Gnamdev/nexusdesk-notification-service) | Notification service | Email / push notifications |
| [`nexusdesk-frontend`](https://github.com/Gnamdev/nexusdesk-frontend) | Frontend | React application |
| [`nexusdesk-infra`](https://github.com/Gnamdev/nexusdesk-infra) | Infrastructure | Docker Compose, Keycloak, DB setup |

## How to run

Each service is a standard Spring Boot Maven project. From a service folder:

```bash
cd nexusdesk-workspace-service
./mvnw spring-boot:run          # or: mvn spring-boot:run
```

Services need the infrastructure running first (PostgreSQL + Keycloak). See the
[`nexusdesk-infra`](https://github.com/Gnamdev/nexusdesk-infra) repository.

## Architecture

See [`NEXUSDESK_BACKEND_BLUEPRINT.md](./NEXUSDESK_BACKEND_BLUEPRINT.md) for the
full backend architecture blueprint.

## Conventions

- **DTO pattern** — never expose JPA entities in API responses.
- **Interface + implementation** — `XService` / `XServiceImpl`.
- **Global exception handler** — `GlobalExceptionHandler` for `BusinessException`,
  `ResourceNotFoundException`, `AccessDeniedException`.
- **Lombok** — `@Data`, `@Builder`, `@RequiredArgsConstructor`.
- **Security** — Spring Security OAuth2 resource server validating JWTs against
  Keycloak.

## License

Proprietary — internal project.