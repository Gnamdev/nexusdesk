# NexusDesk Infrastructure

This directory contains infrastructure setup for the NexusDesk project.

## Docker Compose
We will create a `docker-compose.yml` in `infrastructure/keycloak/` to manage the Keycloak container. This keeps Keycloak separate from the application code.

## Git Strategy
- **Root Repository:** The `nexusdesk` folder is the root.
- **Service Repositories:** Each backend service (e.g., `backend/workspace-service`) will be initialized as its own independent Git repository (a "multi-repo" structure). This allows each team to manage its own lifecycle, CI/CD, and versioning.

## Plan for this turn:
1. **Infrastructure:** Create `infrastructure/keycloak/docker-compose.yml` and `.env.example`.
2. **Git Root:** Finalize the root `.gitignore` to prevent tracking of sensitive files or build artifacts.
3. **Workspace Service:** Initialize it as a separate repository.
4. **PostgreSQL:** I will try to start the Windows Service for Postgres to ensure our local database is ready for the services.

Do you approve this plan?
