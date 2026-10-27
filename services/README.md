# Services

Each folder here becomes an independent Spring Boot application with its own `pom.xml`, `Dockerfile` and database. They are empty on purpose: you generate each one in the phase where it is introduced (see `docs/ROADMAP.md`).

| Service | Phase | Port (local) | Responsibility |
|---|---|---|---|
| `user-service` | 1 | 8081 | Organizations, user profiles, addresses, preferences |
| `listing-service` | 2 | 8082 | Food listings and their lifecycle |
| `config-server` | 3 | 8888 | Serves config from `config-repo/` |
| `discovery-server` | 3 | 8761 | Eureka service registry |
| `api-gateway` | 3 | 8080 | Single entry point, routing, auth, rate limiting |
| `notification-service` | 6 | 8085 | Sends notifications from events |
| `matching-service` | 7 | 8083 | Geo + rule-based matching |
| `pickup-service` | 8 | 8084 | Pickup jobs and the claim→delivery saga |
| `impact-service` | 10 | 8086 | Impact stats (CQRS read model) — stretch goal |

Package root for all services: `com.plateforward.<service>`
