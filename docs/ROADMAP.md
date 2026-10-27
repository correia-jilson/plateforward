# PlateForward — Project Roadmap & Learning Plan

> A learning-first plan for building a Java microservices system, one concept at a time.
> Rule of the project: **you write the code; every phase ends only when you can explain what you built.**

---

## 1. The idea

**PlateForward** is a surplus food rescue network.

Every day, restaurants, bakeries, grocery stores and event caterers throw away food that is still safe to eat, while shelters, food banks and community kitchens nearby struggle to feed people. The problem is rarely the food itself. It is **coordination**: who has what, who needs it, who can carry it, and how fast, before it expires.

PlateForward connects three kinds of people:

| Role | What they do |
|---|---|
| **Donor** (restaurant, store, caterer) | Posts surplus food: what it is, how much, where, and when it expires |
| **Recipient** (shelter, food bank, NGO) | Gets matched with nearby food they can use and claims it |
| **Volunteer** (driver, cyclist) | Accepts a pickup job and carries food from donor to recipient |

A typical flow:

1. A bakery posts "40 loaves of bread, pick up before 9 PM."
2. The system finds recipients within a few kilometres who accept bakery items and notifies them.
3. A shelter claims the listing.
4. The system offers the pickup to nearby volunteers; one accepts.
5. The volunteer marks it picked up, then delivered. Everyone is notified, and the donation is recorded for impact stats (kg of food saved, meals provided).

### Why this project suits microservices

This is not a CRUD app cut into pieces. Its problems genuinely need distributed design:

- **Different parts change at different speeds.** Listings are created constantly; user profiles rarely change. They should scale independently.
- **Things happen asynchronously.** Posting food should not wait for notifications to be sent. That is a natural fit for events (Kafka).
- **Multi-step workflows can fail midway.** Claimed → volunteer assigned → picked up → delivered. If no volunteer accepts, the claim must be released. That is the **Saga pattern**.
- **Time and location matter.** Expiry deadlines and geo-matching make it more than form-filling.

### Resume summary (what you will be able to claim)

> Designed and built **PlateForward**, an event-driven food rescue platform using **Java, Spring Boot and Spring Cloud** microservices with an **API Gateway, service discovery, centralized config, Kafka** messaging, **Keycloak** OAuth2 security, **PostgreSQL/PostGIS** geo-matching, **Resilience4j** fault tolerance, **OpenTelemetry** tracing, and a **Next.js** frontend; containerized with **Docker** and deployed to **Kubernetes** with **GitHub Actions** CI/CD.

---

## 2. Architecture overview

```
                         ┌───────────────────┐
                         │  Next.js frontend  │
                         └─────────┬─────────┘
                                   │ HTTPS (JWT)
                         ┌─────────▼─────────┐        ┌──────────────┐
                         │    API Gateway     │◄──────►│   Keycloak   │
                         │ (Spring Cloud GW)  │        │ (auth/roles) │
                         └─────────┬─────────┘        └──────────────┘
          ┌──────────────┬─────────┼──────────┬───────────────┐
          ▼              ▼         ▼          ▼               ▼
   ┌────────────┐ ┌────────────┐ ┌──────────┐ ┌────────────┐ ┌──────────────┐
   │   user-    │ │  listing-  │ │ matching-│ │  pickup-   │ │ notification-│
   │  service   │ │  service   │ │ service  │ │  service   │ │   service    │
   └─────┬──────┘ └─────┬──────┘ └────┬─────┘ └─────┬──────┘ └──────┬───────┘
         │              │             │             │               │
     [Postgres]   [Postgres+PostGIS] [Redis]   [Postgres]       [Postgres]
         │              │             │             │               │
         └──────────────┴──────┬──────┴─────────────┴───────────────┘
                               ▼
                    ┌─────────────────────┐
                    │   Kafka (events)    │
                    └─────────────────────┘

  Supporting: config-server · discovery-server (Eureka) · Prometheus · Grafana · Zipkin/Tempo
```

### The services

| Service | Responsibility | Owns data | Talks via |
|---|---|---|---|
| **api-gateway** | Single entry point: routing, JWT validation, rate limiting, CORS | — | HTTP |
| **discovery-server** | Eureka registry so services find each other by name | — | HTTP |
| **config-server** | Serves config for every service from `config-repo/` | — | HTTP |
| **user-service** | Profiles for donors, recipients, volunteers; organizations; addresses; preferences | users, organizations | REST |
| **listing-service** | Food listings: create, update, expire, claim; geo-location | listings, claims | REST + publishes events |
| **matching-service** | Finds recipients/volunteers near a listing using rules (distance, food type, capacity) | match cache (Redis) | Consumes + publishes events |
| **pickup-service** | Pickup jobs and their lifecycle; runs the claim→pickup→delivery saga | pickups | REST + events |
| **notification-service** | Email / push / in-app notifications | notification log | Consumes events |
| **impact-service** *(stretch)* | Stats: kg saved, meals, CO₂ avoided; dashboards | read-optimized stats (CQRS) | Consumes events |

### Key architecture principles we will follow

1. **Database per service.** No service reads another service's tables. Ever.
2. **Synchronous calls only when you need an answer now** (e.g. "does this user exist?"); everything else is an **event**.
3. **Events are facts in past tense:** `ListingCreated`, `ListingClaimed`, `PickupAssigned`, `PickupDelivered`.
4. **Every service is independently buildable, testable, and deployable.**
5. **Design for failure:** timeouts, retries, circuit breakers, idempotent consumers.

---

## 3. Tech stack

| Area | Choice | Why |
|---|---|---|
| Language | Java (current LTS — confirm version in Phase 0) | Industry standard for microservices |
| Framework | Spring Boot + Spring Cloud | Most in-demand Java stack |
| Build | Maven (one `pom.xml` per service) | Widely used, simple to read |
| Database | PostgreSQL (+ PostGIS for location) | Reliable, has geo support |
| Migrations | Flyway | Versioned, reviewable schema changes |
| Cache | Redis | Fast lookups, distributed locks |
| Messaging | Apache Kafka | Event-driven backbone |
| Auth | Keycloak (OAuth2 / OpenID Connect) | Real-world identity management |
| Resilience | Resilience4j | Circuit breakers, retries, rate limits |
| Observability | Micrometer, Prometheus, Grafana, OpenTelemetry, Zipkin | Metrics + distributed tracing |
| Testing | JUnit 5, Mockito, Testcontainers | Real databases/Kafka in tests |
| API docs | springdoc-openapi (Swagger UI) | Auto-generated API docs |
| Frontend | Next.js (React, TypeScript) + Tailwind | Simple, popular, good for maps/forms |
| Containers | Docker, Docker Compose | Run everything locally |
| Orchestration | Kubernetes (kind or minikube locally) | Production-style deployment |
| CI/CD | GitHub Actions | Build/test on every push |

> Exact versions are decided at the start of Phase 1 by checking what is current. We pin them and write them in the README.

---

## 4. Repository structure

We use a **monorepo**: one Git repository holding every service. Each service is still fully independent (own `pom.xml`, own Dockerfile, own database).

```
plateforward/
├── README.md                     # Project overview and how to run it
├── docs/
│   ├── ROADMAP.md                # This file
│   ├── architecture/             # Diagrams, event catalog, API contracts
│   ├── adr/                      # Architecture Decision Records (why we chose X)
│   └── learning-notes/           # Your own notes per phase
├── services/
│   ├── api-gateway/
│   ├── discovery-server/
│   ├── config-server/
│   ├── user-service/
│   ├── listing-service/
│   ├── matching-service/
│   ├── pickup-service/
│   ├── notification-service/
│   └── impact-service/           # stretch goal
├── libs/
│   └── common-events/            # Shared event classes (kept minimal on purpose)
├── config-repo/                  # YAML config served by config-server
├── frontend/                     # Next.js app
├── infrastructure/
│   ├── docker/                   # docker-compose files, init scripts
│   ├── k8s/                      # Kubernetes manifests
│   └── monitoring/               # Prometheus, Grafana config
└── .github/
    └── workflows/                # CI pipelines
```

### Inside every Java service (you will create this in Phase 1)

We use **package-by-layer** inside each service at first, because it is the easiest to learn. Later you may refactor to package-by-feature.

```
listing-service/
├── pom.xml
├── Dockerfile
└── src/
    ├── main/
    │   ├── java/com/plateforward/listing/
    │   │   ├── ListingServiceApplication.java   # main class
    │   │   ├── controller/      # REST endpoints (HTTP in/out only)
    │   │   ├── service/         # business logic
    │   │   ├── repository/      # Spring Data JPA interfaces
    │   │   ├── domain/          # JPA entities + enums (the "model")
    │   │   ├── dto/             # request/response objects (never expose entities)
    │   │   ├── mapper/          # entity ↔ DTO conversion
    │   │   ├── event/           # Kafka producers/consumers and event classes
    │   │   ├── client/          # calls to other services (RestClient/Feign)
    │   │   ├── config/          # Spring configuration beans
    │   │   └── exception/       # custom exceptions + global handler
    │   └── resources/
    │       ├── application.yml
    │       └── db/migration/    # Flyway SQL: V1__create_listing.sql ...
    └── test/
        └── java/com/plateforward/listing/
            ├── controller/      # web layer tests (@WebMvcTest)
            ├── service/         # unit tests (Mockito)
            └── integration/     # Testcontainers tests
```

**The one rule to remember:** a request flows `controller → service → repository`. Controllers never touch repositories directly, and entities never leave the service layer (DTOs do).

---

## 5. Phases

Each phase has: **Goal → Concepts → What you build → Done when → Can you explain?**
Do not move on until you can answer the "Can you explain?" questions in your own words. Write the answers in `docs/learning-notes/phase-N.md`.

---

### Phase 0 — Foundations & setup
**Goal:** Tools installed, repo on GitHub, and you understand *why* microservices.

**Concepts:** monolith vs microservices; bounded contexts; when microservices are a bad idea; monorepo vs polyrepo; Git branching.

**Build:**
- Install JDK (LTS), Maven, Docker Desktop, IntelliJ IDEA (or VS Code), Node.js, Git.
- Push this repo to GitHub.
- Write your first ADR: `docs/adr/0001-use-monorepo.md`.
- Draw the domain on paper: entities (Listing, Organization, Pickup...) and which service owns each.

**Done when:** repo is on GitHub; `docker run hello-world` works; your domain sketch is in `docs/architecture/`.

**Can you explain?**
- What problem do microservices solve, and what new problems do they create?
- Why does each service own its own database?

---

### Phase 1 — Your first service: `user-service`
**Goal:** Build one excellent Spring Boot service before building many.

**Concepts:** Spring Boot auto-configuration; dependency injection; REST design (verbs, status codes); JPA/Hibernate; Flyway migrations; DTOs and validation; global exception handling; unit vs integration tests; Testcontainers.

**Build:**
- Generate the project with Spring Initializr (Web, Data JPA, Validation, PostgreSQL, Flyway, Actuator, Lombok optional).
- Entities: `Organization` (type: DONOR / RECIPIENT), `UserProfile` (role: DONOR_STAFF / RECIPIENT_STAFF / VOLUNTEER / ADMIN), `Address` (with lat/long).
- Endpoints: CRUD for organizations and profiles, paging and filtering.
- Postgres via `infrastructure/docker/docker-compose.yml`.
- Swagger UI via springdoc.
- Tests: service unit tests + one Testcontainers integration test.

**Done when:** you can create/read/update organizations through Swagger; tests pass with `mvn verify`.

**Can you explain?**
- What happens, step by step, when an HTTP request hits your controller?
- Why use DTOs instead of returning entities?
- Why Flyway instead of `ddl-auto: update`?

---

### Phase 2 — Second service & service-to-service calls: `listing-service`
**Goal:** Feel the pain (and benefit) of two services that depend on each other.

**Concepts:** database per service; synchronous HTTP between services (`RestClient` / OpenFeign); handling a dependency being down; listing lifecycle as a state machine; timestamps and time zones.

**Build:**
- `FoodListing` entity: title, category (BAKERY, PRODUCE, COOKED_MEAL, DAIRY...), quantity + unit, pickup window, expiry time, location, status (AVAILABLE → CLAIMED → PICKED_UP → DELIVERED / EXPIRED / CANCELLED).
- On create, call `user-service` to verify the donor organization exists.
- Scheduled job that marks expired listings.
- Second Postgres database in Docker Compose.

**Done when:** creating a listing for a fake donor fails cleanly; stopping `user-service` gives a sensible error, not a stack trace.

**Can you explain?**
- What happens to `listing-service` when `user-service` is slow? Why is that dangerous?
- Why are state transitions enforced in the service layer?

---

### Phase 3 — Infrastructure services: config, discovery, gateway
**Goal:** Turn two services into a *system*.

**Concepts:** centralized configuration; service discovery and client-side load balancing; API Gateway pattern; routing and filters; profiles (`local`, `docker`).

**Build:**
- `config-server` serving YAML from `config-repo/`.
- `discovery-server` (Eureka); both services register.
- `api-gateway` routes `/api/users/**` and `/api/listings/**`.
- Replace hard-coded URLs with service names.
- Run two instances of `listing-service` and watch load balancing.

**Done when:** everything runs with one `docker compose up`, and clients only ever call the gateway.

**Can you explain?**
- Why not let the frontend call each service directly?
- How does `listing-service` find `user-service` without knowing its IP?

---

### Phase 4 — Security with Keycloak
**Goal:** Real authentication and role-based authorization.

**Concepts:** OAuth2 and OpenID Connect; access tokens vs ID tokens; JWT structure and validation; resource servers; validating at the gateway *and* the service; roles and ownership checks.

**Build:**
- Keycloak in Docker Compose with a `plateforward` realm and roles: `DONOR`, `RECIPIENT`, `VOLUNTEER`, `ADMIN`.
- Gateway and services configured as OAuth2 resource servers.
- Rules: only donors create listings; only recipients claim; only the owning donor edits their listing.
- Link Keycloak user ID to `UserProfile`.

**Done when:** requests without a token get 401; wrong role gets 403; a donor cannot edit another donor's listing.

**Can you explain?**
- What is inside a JWT, and why can a service trust it without calling Keycloak every time?
- Difference between authentication and authorization?

---

### Phase 5 — Frontend with Next.js
**Goal:** A usable app, so the project is demo-able.

**Concepts:** Next.js App Router; server vs client components; login with Keycloak (NextAuth/Auth.js); calling the gateway with tokens; forms and validation; maps (Leaflet + OpenStreetMap).

**Build:**
- Pages: login, donor dashboard (post/manage listings), recipient feed (nearby listings on a map + claim), volunteer jobs page, profile.
- Typed API client for the gateway.

**Done when:** a donor can post food and a recipient can claim it, entirely through the UI.

**Can you explain?**
- How does the browser get a token, and how does it reach your Java services?

---

### Phase 6 — Event-driven architecture with Kafka
**Goal:** Decouple services using events.

**Concepts:** topics, partitions, consumer groups, offsets; at-least-once delivery; idempotent consumers; event schema design and versioning; the **Transactional Outbox** pattern (why "save to DB then publish" can lose events).

**Build:**
- Kafka in Docker Compose.
- `listing-service` publishes `ListingCreated`, `ListingClaimed`, `ListingExpired` (via an outbox table).
- `notification-service` consumes them and records/sends notifications (email via a local tool like Mailpit).
- `libs/common-events` holds the event classes.
- Write `docs/architecture/event-catalog.md`.

**Done when:** posting a listing triggers a notification without `listing-service` knowing `notification-service` exists.

**Can you explain?**
- What happens if `notification-service` is down for an hour?
- Why might the same event be processed twice, and how do you make that safe?

---

### Phase 7 — Matching with geo-queries & Redis
**Goal:** The "smart" part of the product.

**Concepts:** PostGIS and spatial indexes; distance queries; rule-based matching; caching strategies (cache-aside, TTL); Redis data types.

**Build:**
- `matching-service` consumes `ListingCreated`, finds recipients within radius X who accept that category, and publishes `MatchesFound`.
- Recipient preferences (categories, max distance, capacity) in `user-service`.
- Cache recipient preferences in Redis; invalidate on `PreferencesUpdated`.

**Done when:** a listing in one neighbourhood only notifies recipients nearby who want that food type.

**Can you explain?**
- Why is a spatial index needed, and what does it do?
- When does a cache make things worse?

---

### Phase 8 — Pickups and the Saga pattern
**Goal:** Handle a multi-step, multi-service workflow that can fail.

**Concepts:** distributed transactions and why 2PC is avoided; Saga (choreography vs orchestration); compensating actions; timeouts as events; distributed locks (Redis) to stop two volunteers taking one job.

**Build:**
- `pickup-service` with `PickupJob` (OFFERED → ACCEPTED → PICKED_UP → DELIVERED / FAILED).
- Saga: `ListingClaimed` → create job → offer to volunteers → if nobody accepts before deadline, publish `PickupFailed` → `listing-service` releases the claim (compensation).
- Volunteer "accept job" guarded by a lock.

**Done when:** two volunteers accepting simultaneously results in exactly one assignment; an unaccepted job correctly un-claims the listing.

**Can you explain?**
- Choreography vs orchestration — which did you use and why?
- What is a compensating action, and why isn't it the same as a rollback?

---

### Phase 9 — Resilience
**Goal:** The system survives partial failure.

**Concepts:** timeouts; retries with backoff; circuit breakers; bulkheads; rate limiting at the gateway; graceful degradation; dead-letter topics.

**Build:**
- Resilience4j on every synchronous call.
- Gateway rate limiting (Redis-backed).
- Kafka retry + dead-letter topic for poison messages.
- A "chaos" exercise: kill services one by one and record what happens in your notes.

**Done when:** killing any single non-gateway service never takes down the others.

**Can you explain?**
- Why can retries make an outage worse?
- What are the three states of a circuit breaker?

---

### Phase 10 — Observability
**Goal:** See inside the running system.

**Concepts:** the three pillars (logs, metrics, traces); correlation IDs; structured JSON logging; RED metrics; distributed tracing with OpenTelemetry.

**Build:**
- Actuator + Micrometer → Prometheus → Grafana dashboard per service.
- Tracing: follow one listing from gateway → listing → Kafka → matching → notification in Zipkin/Tempo.
- Centralized logs (Loki or ELK — pick one).
- `impact-service` (stretch): CQRS read model for kg saved / meals provided, shown in the UI.

**Done when:** you can find why a specific request was slow using only dashboards and traces.

---

### Phase 11 — Containers, Kubernetes & CI/CD
**Goal:** Deploy it like a real team would.

**Concepts:** multi-stage Docker builds; image size and layers; Kubernetes Deployments, Services, ConfigMaps, Secrets, probes; health checks; CI pipelines.

**Build:**
- Optimized Dockerfile per service.
- Kubernetes manifests (or a Helm chart) in `infrastructure/k8s/`, run on kind/minikube.
- GitHub Actions: build + test only the services that changed; build images on main.
- Optional: deploy a demo to a cloud free tier.

**Done when:** a push to `main` runs tests automatically, and the whole system runs on a local Kubernetes cluster.

---

## 6. How we work together

1. **Start each phase by reading this file's section for it** and asking questions before writing code.
2. **I explain, you type.** I'll explain the concept, show small examples, and review your code. I won't write the services for you.
3. **One branch per phase:** `phase-1-user-service`, merged to `main` with a pull request you review yourself.
4. **Commit small and often** with clear messages (`feat(listing): add expiry scheduler`).
5. **Write learning notes** at the end of each phase in `docs/learning-notes/`.
6. **Record big decisions** as ADRs in `docs/adr/`.

## 7. Git conventions

- Branches: `phase-N-short-name`, `feat/...`, `fix/...`
- Commit style: Conventional Commits — `feat:`, `fix:`, `docs:`, `test:`, `refactor:`, `chore:`
- `main` is always runnable.

## 8. Progress tracker

| Phase | Topic | Status |
|---|---|---|
| 0 | Foundations & setup | ⬜ |
| 1 | user-service | ⬜ |
| 2 | listing-service + sync calls | ⬜ |
| 3 | Config, discovery, gateway | ⬜ |
| 4 | Keycloak security | ⬜ |
| 5 | Next.js frontend | ⬜ |
| 6 | Kafka events + notifications | ⬜ |
| 7 | Matching, PostGIS, Redis | ⬜ |
| 8 | Pickups + Saga | ⬜ |
| 9 | Resilience | ⬜ |
| 10 | Observability (+ impact-service) | ⬜ |
| 11 | Docker, Kubernetes, CI/CD | ⬜ |
