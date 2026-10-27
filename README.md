# PlateForward

**A surplus food rescue network built with Java microservices.**

Restaurants, bakeries and grocery stores throw away safe, edible food every day, while shelters and food banks nearby need it. PlateForward connects them: donors post surplus food, nearby recipients are matched and claim it, and volunteers pick it up and deliver it before it expires.

## Status

🚧 Learning project in progress. See [docs/ROADMAP.md](docs/ROADMAP.md) for the full plan and phase tracker.

## Architecture at a glance

- **Backend:** Java, Spring Boot, Spring Cloud (Gateway, Config, Eureka)
- **Services:** user, listing, matching, pickup, notification, impact
- **Data:** PostgreSQL + PostGIS, Redis
- **Messaging:** Apache Kafka
- **Security:** Keycloak (OAuth2 / OIDC)
- **Frontend:** Next.js
- **Ops:** Docker, Kubernetes, GitHub Actions, Prometheus, Grafana, OpenTelemetry

## Repository layout

| Folder | Contents |
|---|---|
| `services/` | One folder per microservice, each independently buildable |
| `libs/` | Small shared libraries (event definitions) |
| `config-repo/` | Centralized configuration served by `config-server` |
| `frontend/` | Next.js web app |
| `infrastructure/` | Docker Compose, Kubernetes manifests, monitoring config |
| `docs/` | Roadmap, architecture, decision records, learning notes |

## Running locally

Instructions will be added as each phase is completed.
