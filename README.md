# FitForMe — AI-Powered Fitness Tracking Application

A full-stack **microservices** fitness application: users log workouts, and an AI service (Google Gemini) analyzes each activity and produces personalized recommendations. Built with **Java Spring Boot / Spring Cloud** on the backend and **React** on the frontend.

> This repository is based on the *Java Spring Boot AI Full Stack Microservices Course* by [EmbarkX](http://www.embarkx.com) (instructor: Faisal Memon). See [Credits](#credits) at the bottom.

---

## Table of Contents

- [Features](#features)
- [Architecture](#architecture)
- [Services Overview](#services-overview)
- [Key Workflows](#key-workflows)
- [Technology Stack](#technology-stack)
- [API Reference](#api-reference)
- [Data Models](#data-models)
- [Security](#security)
- [Configuration](#configuration)
- [Prerequisites](#prerequisites)
- [Building the Project](#building-the-project)
- [Running the Application](#running-the-application)
- [Frontend](#frontend)
- [Project Structure](#project-structure)
- [Testing](#testing)
- [Known Issues & Limitations](#known-issues--limitations)
- [Credits](#credits)

---

## Features

- **User management** — self-registration, profile lookup, and user-existence validation backed by PostgreSQL.
- **Activity tracking** — log workouts (type, duration, calories, metrics) with per-user history, stored in MongoDB.
- **AI recommendations** — every logged activity is published to RabbitMQ; the AI service calls **Google Gemini** and stores an analysis (improvements, suggestions, safety guidelines) in MongoDB.
- **Single entry point** — an API Gateway routes and load-balances all traffic, validates JWTs, and auto-syncs Keycloak users into the local user database.
- **Service discovery & centralized config** — Netflix Eureka registry + Spring Cloud Config Server.
- **OAuth2 login** — the SPA uses Authorization Code + PKCE against Keycloak.

---

## Architecture

```
                      ┌───────────────────────────────────────┐
                      │            Keycloak :8181             │
                      │   realm: fitness-oauth2 (OIDC/OAuth2) │
                      └────────────────▲──────────────────────┘
                                       │  login / JWT (PKCE)
                                       │
 ┌──────────────────┐   HTTP :8080/api │    ┌─────────────────────────────┐
 │  React SPA :5173 │ ────────────────►│───▶│   API Gateway (8080)        │
 │  (Vite build)    │  Authorization:  │    │  Spring Cloud Gateway       │
 │                  │  Bearer <JWT>    │    │  • OAuth2 resource server   │
 │  + X-User-ID     │◄────────────────│    │  • KeycloakUserSyncFilter   │
 └──────────────────┘                  │    │  • CORS for :5173           │
                                       │    └──────┬───────┬──────┬───────┘
                                       │           │       │      │  lb:// via Eureka
                                       │   ┌───────┘       │      └────────┐
                                       │   ▼               ▼              ▼
                                       │ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
                                       │ │user-service  │ │activity-svc  │ │ ai-service   │
                                       │ │   :8081      │ │   :8082      │ │   :8083      │
                                       │ │ PostgreSQL   │ │ MongoDB      │ │ MongoDB      │
                                       │ │fitness_user_db│ │fitnessactivity│ │fitnessrecomm.│
                                       │ └──────▲───────┘ └──────┬───────┘ └──────▲───────┘
                                       │        │ REST            │ RabbitMQ       │
                                       │        │ validate/       │ fitness.exchange│
                                       │        │ register        │ activity.tracking
                                       │        └─────────────────┼───────────────│
                                       │                          ▼               │
                                       │                   activity.queue ───────▶│
                                       │                          (consumer)      │
                                       │                                          │
                                       │                              Google Gemini API
                                       │                              GEMINI_API_URL/KEY
                                       │
   Supporting services:  Config Server :8888 (native, classpath:/config)
                         Eureka       :8761 (service registry)
```

**Communication patterns**

| Pattern | Where | Technology |
|---|---|---|
| Browser → gateway | SPA calls `http://localhost:8080/api/**` | axios + Bearer JWT |
| Synchronous service-to-service | gateway & activity-service → user-service | `@LoadBalanced WebClient` (`http://USER-SERVICE`) |
| Asynchronous messaging | activity-service → ai-service | RabbitMQ (`fitness.exchange` → `activity.queue`) |
| Discovery | all services → registry | Eureka client/server |
| Configuration | all services → config server | Spring Cloud Config (`spring.config.import`) |

---

## Services Overview

| Service | Port | Java | Role | Storage / Tech |
|---|---|---|---|---|
| **configserver** | `8888` | 17 | Centralized configuration (native profile, serves `classpath:/config/*.yml`) | — |
| **eureka** | `8761` | 17 | Service registry (standalone; doesn't register itself) | — |
| **gateway** | `8080` | 23 | API entry point: routing, JWT validation, CORS, Keycloak → local user sync | Spring Cloud Gateway (WebFlux) |
| **userservice** | `8081` | 23 | User registration, profile, existence validation | PostgreSQL (`fitness_user_db`) via JPA/Hibernate |
| **activityservice** | `8082` | 23 | Activity CRUD-lite, validates user, publishes activities to the queue | MongoDB (`fitnessactivity`) + RabbitMQ producer |
| **aiservice** | `8083` | 23 | Consumes activities, calls Gemini, stores & serves recommendations | MongoDB (`fitnessrecommendation`) + RabbitMQ consumer + Gemini |
| **fitness-app-frontend** | `5173` (dev) | — | React SPA (activity logging + AI recommendations) | Vite build → `dist/` |

**Gateway routes** (defined in `configserver/src/main/resources/config/api-gateway.yml`):

| Path predicate | Target (Eureka) |
|---|---|
| `/api/users/**` | `lb://USER-SERVICE` |
| `/api/activities/**` | `lb://ACTIVITY-SERVICE` |
| `/api/recommendations/**` | `lb://AI-SERVICE` |

---

## Key Workflows

### 1. Authentication & user auto-sync
1. The SPA redirects to Keycloak (`fitness-oauth2` realm, client `oauth2-pkce-client`) using Authorization Code + PKCE.
2. On every API call, the axios interceptor attaches `Authorization: Bearer <JWT>` and `X-User-ID` (JWT `sub`).
3. The gateway's `SecurityConfig` validates the JWT against Keycloak's JWKS endpoint.
4. `KeycloakUserSyncFilter` (a `WebFilter`) parses the token (Nimbus), calls `GET /api/users/{id}/validate` on user-service, and **auto-registers** the user (`POST /api/users/register`) if unknown.
5. The filter injects `X-User-ID` into the downstream request — the identity header all services rely on.

### 2. Logging an activity
1. `POST /api/activities` (with `X-User-ID`) hits activity-service.
2. `UserValidationService` re-validates the user against user-service via load-balanced WebClient.
3. The activity is saved to MongoDB, then published as JSON to RabbitMQ exchange **`fitness.exchange`** with routing key **`activity.tracking`** (queue `activity.queue`).
4. Failures to publish are caught and logged (the activity is still saved).

### 3. AI recommendation
1. ai-service's `@RabbitListener(queues = "activity.queue")` receives the activity.
2. `ActivityAIService` builds a prompt (activity type, duration, calories, metrics) requesting a strict JSON schema and `GeminiService` POSTs it to `${GEMINI_API_URL}?key=${GEMINI_API_KEY}`.
3. The response is parsed (`candidates[0].content.parts[0].text`, Markdown fences stripped) into a `Recommendation` (`analysis`, `improvements`, `suggestions`, `safety`) and saved to MongoDB.
4. On any parse failure a **fallback default recommendation** is stored instead.
5. The SPA shows it via `GET /api/recommendations/activity/{id}` on the activity detail page.

---

## Technology Stack

**Backend**
- Java 17 / 23 (see [Building the Project](#building-the-project)), Spring Boot **3.4.3**, Spring Cloud **2024.0.0**
- Spring Cloud Gateway, Eureka, Config Server, Spring Web / WebFlux, Spring Data JPA + MongoDB, Spring AMQP
- Spring Security OAuth2 Resource Server (JWT), Nimbus JWT, WebClient (load-balanced), Google Gemini API
- PostgreSQL, MongoDB, RabbitMQ, Lombok, Maven Wrapper (`./mvnw`)

**Frontend**
- React 19 + Vite 6, react-router v7, MUI v6 (`sx` styling), Redux Toolkit, axios, `react-oauth2-code-pkce`, ESLint 9

---

## API Reference

> All routes go through the gateway: `http://localhost:8080/api/...` and require a valid Bearer token (except as noted).

### user-service (`/api/users`)
| Method | Path | Body / Params | Response |
|---|---|---|---|
| `POST` | `/api/users/register` | `RegisterRequest` `{ email, password, keycloakId, firstName, lastName }` | `UserResponse` |
| `GET` | `/api/users/{userId}` | — | `UserResponse` |
| `GET` | `/api/users/{userId}/validate` | — | `Boolean` |

### activity-service (`/api/activities`)
| Method | Path | Headers / Body | Response |
|---|---|---|---|
| `POST` | `/api/activities` | Header `X-User-ID` (required), `ActivityRequest` `{ type, duration, caloriesBurned, startTime?, additionalMetrics? }` | `ActivityResponse` |
| `GET` | `/api/activities` | Header `X-User-ID` | `List<ActivityResponse>` |
| `GET` | `/api/activities/{activityId}` | — | `ActivityResponse` |

`type` ∈ `RUNNING, WALKING, CYCLING, SWIMMING, WEIGHT_TRAINING, YOGA, HIIT, CARDIO, STRETCHING, OTHER`

### ai-service (`/api/recommendations`)
| Method | Path | Response |
|---|---|---|
| `GET` | `/api/recommendations/activity/{activityId}` | `Recommendation` |
| `GET` | `/api/recommendations/user/{userId}` | `List<Recommendation>` |

### Example

```bash
# Start the flow with a token obtained from Keycloak, then:
curl -X POST http://localhost:8080/api/activities \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-User-ID: <keycloak-sub>" \
  -H "Content-Type: application/json" \
  -d '{"type":"RUNNING","duration":30,"caloriesBurned":320}'
```

---

## Data Models

**`users` (PostgreSQL, JPA)** — `User`: `id` (UUID), `email` (unique), `keycloakId`, `password`, `firstName`, `lastName`, `role` (`USER`/`ADMIN`), `createdAt`, `updatedAt`.

**`activities` (MongoDB)** — `Activity`: `id`, `userId`, `type` (enum), `duration`, `caloriesBurned`, `startTime`, `additionalMetrics` (stored as `metrics`), `createdAt`, `updatedAt` (`@EnableMongoAuditing`).

**`recommendations` (MongoDB)** — `Recommendation`: `id`, `activityId`, `userId`, `activityType`, `recommendation` (analysis text), `improvements[]`, `suggestions[]`, `safety[]`, `createdAt`.

**RabbitMQ** — durable `DirectExchange` `fitness.exchange`, routing key `activity.tracking`, durable queue `activity.queue`, `Jackson2JsonMessageConverter` on both sides.

---

## Security

- **Gateway is the only secured hop.** `@EnableWebFluxSecurity` with `oauth2ResourceServer().jwt()`; JWTs validated via Keycloak JWKS (`.../realms/fitness-oauth2/protocol/openid-connect/certs`); CSRF disabled; every exchange requires authentication.
- **CORS** allows origin `http://localhost:5173`, methods `GET/POST/PUT/DELETE/OPTIONS`, headers `Authorization, Content-Type, X-User-ID`, with credentials.
- **Downstream services (8081–8083) have no Spring Security.** They trust the `X-User-ID` header injected by the gateway; activity-service additionally re-validates the user against user-service. Treat this as a course-level simplification — see [Known Issues](#known-issues--limitations).
- **User data protection**: registration is idempotent (existing e-mail returns the existing user); passwords are stored but not used for local login (authentication happens entirely in Keycloak).

---

## Configuration

All runtime configuration for the gateway and business services is served by the **config server** from `configserver/src/main/resources/config/*.yml` (native/classpath profile):

| File | Service | Notable settings |
|---|---|---|
| `user-service.yml` | userservice | `jdbc:postgresql://localhost:5432/fitness_user_db` (postgres / admin@123), `ddl-auto: update` |
| `activity-service.yml` | activityservice | Mongo `fitnessactivity`, RabbitMQ guest/guest, exchange/queue/routing-key names |
| `ai-service.yml` | aiservice | Mongo `fitnessrecommendation`, RabbitMQ, `gemini.api.url/key` from env |
| `api-gateway.yml` | gateway | Port, routes, `jwk-set-uri` |

**Environment variables**

| Variable | Used by | Purpose |
|---|---|---|
| `GEMINI_API_URL` | aiservice | Gemini endpoint URL (required to generate real recommendations) |
| `GEMINI_API_KEY` | aiservice | Gemini API key |

> The `spring.config.import: optional:configserver:...` entries are **optional**, so services boot even if the config server is down — but they will not receive ports/routes/credentials from it.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| **JDK 17+** (17, 22 or 23) | `configserver`/`eureka` target 17; the other four services declare `java.version=23`. See [Building the Project](#building-the-project) if you only have JDK 17 or newer-but-not-23. |
| **Maven** | Not required — every service ships the Maven Wrapper (`mvnw` / `mvnw.cmd`). |
| **Node.js 18+ / npm** | For the frontend. |
| **PostgreSQL** | Port `5432`, database `fitness_user_db`, user `postgres` / password `admin@123` (or edit `user-service.yml`). |
| **MongoDB** | Port `27017`; databases `fitnessactivity` and `fitnessrecommendation` are created automatically. |
| **RabbitMQ** | Port `5672`, default `guest`/`guest`. |
| **Keycloak** | Port `8181`, realm `fitness-oauth2`, client `oauth2-pkce-client` with redirect URI `http://localhost:5173`. |
| **Gemini API key** | For AI recommendations (`GEMINI_API_KEY` / `GEMINI_API_URL`). |

---

## Building the Project

Each service is an independent Maven project (no aggregator POM). From the repository root:

```powershell
# Backend — build one service (repeat for each: configserver, eureka, gateway,
# userservice, activityservice, aiservice)
cd <service>
./mvnw.cmd -B clean package          # Unix: ./mvnw -B clean package
# → target/<service>-0.0.1-SNAPSHOT.jar
```

```bash
# Frontend
cd fitness-app-frontend
npm install
npm run build        # → dist/
```

**JDK notes (important):**

- Four services (`gateway`, `userservice`, `activityservice`, `aiservice`) declare `<java.version>23</java.version>`.
- **If you have JDK 23** — plain `./mvnw clean package` works as-is.
- **If you only have a newer JDK (24+)** — Spring Boot 3.4.3 pins Lombok `1.18.36`, which crashes on newer javac, and JDK 23+ disables implicit annotation processing (breaks `aiservice`/`gateway`, whose poms don't declare Lombok's `annotationProcessorPaths`). Build with:

  ```powershell
  $env:JAVA_HOME = 'C:\path\to\jdk-23-or-newer'
  .\mvnw.cmd -B clean package `
    '-Dlombok.version=1.18.48' `
    '-Dmaven.compiler.proc=full'
  ```
  (bytecode still targets Java 23 because of `--release 23`)

---

## Running the Application

Start in this order (or use your IDE to run all `*Application` classes):

1. **Infrastructure:** PostgreSQL, MongoDB, RabbitMQ, Keycloak (`:8181`).
2. **Config server:** `java -jar configserver/target/configserver-0.0.1-SNAPSHOT.jar` → `http://localhost:8888`
3. **Eureka:** `java -jar eureka/target/eureka-0.0.1-SNAPSHOT.jar` → `http://localhost:8761`
4. **Business services** (any order): `userservice`, `activityservice` (set `GEMINI_API_URL`/`GEMINI_API_KEY` for `aiservice`), `gateway`.
5. **Frontend:** `cd fitness-app-frontend && npm run dev` → `http://localhost:5173`

**Health check:** open `http://localhost:8761` — `USER-SERVICE`, `ACTIVITY-SERVICE`, `AI-SERVICE`, and `API-GATEWAY` should be *UP*, then visit `http://localhost:5173` and log in.

| URL | What |
|---|---|
| `http://localhost:5173` | Frontend SPA |
| `http://localhost:8080/api/...` | API gateway (all backend calls) |
| `http://localhost:8761` | Eureka dashboard |
| `http://localhost:8888` | Config server (`/user-service/default` etc.) |
| `http://localhost:8181` | Keycloak |

---

## Frontend

```
fitness-app-frontend/src/
├── main.jsx               # AuthProvider → Redux Provider → App
├── App.jsx                # login gate, route table, layout
├── authConfig.js          # Keycloak PKCE config (realm/client/endpoints)
├── store/                 # Redux Toolkit: store.js, authSlice.js
├── services/api.js        # axios instance + interceptors + endpoint fns
└── components/            # ActivityForm, ActivityList, ActivityDetail
```

| Route | Page | Notes |
|---|---|---|
| `/` | Login welcome screen / redirect | Shows LOGIN when no token |
| `/activities` | `ActivityForm` + `ActivityList` | Log a workout, browse cards |
| `/activities/:id` | `ActivityDetail` | Activity data + AI recommendation (analysis, improvements, suggestions, safety) |

- **State:** Redux holds only auth (`token`, `user`, `userId`) mirrored to `localStorage`; data fetching is local `useState`/`useEffect`.
- **API layer:** single axios instance with `baseURL: http://localhost:8080/api`; request interceptor attaches `Authorization` and `X-User-ID`.
- **Auth config:** `authConfig.js` → realm `fitness-oauth2`, client `oauth2-pkce-client`, redirect `http://localhost:5173`, scopes `openid profile email offline_access`.
- Scripts: `npm run dev` (Vite, port 5173), `npm run build`, `npm run preview`, `npm run lint`.

---

## Project Structure

```
fitness-app-microservices/
├── configserver/          # Config Server (:8888) + config/*.yml for all services
├── eureka/                # Eureka registry (:8761)
├── gateway/               # API Gateway (:8080) — security, CORS, Keycloak sync
├── userservice/           # User accounts (:8081, PostgreSQL)
├── activityservice/       # Activity tracking (:8082, MongoDB + RabbitMQ producer)
├── aiservice/             # AI recommendations (:8083, MongoDB + RabbitMQ consumer + Gemini)
└── fitness-app-frontend/  # React SPA (Vite)
```

Each backend service follows the same layout: `controller → service → repository → model (+ dto)`, with `config/` holding WebClient, RabbitMQ, Mongo, and security configuration.

---

## Testing

```powershell
cd <service>
.\mvnw.cmd -B test
```

- `configserver` and `eureka` tests are self-contained and **pass without infrastructure**.
- The other services' `@SpringBootTest` context tests boot the **full application context**, which requires the config server (and for userservice a reachable PostgreSQL, for ai/activity services MongoDB + RabbitMQ). Run them with the stack up, or skip with `-DskipTests`.

---

## Known Issues & Limitations

**Backend**
- No `@ControllerAdvice`/`@ExceptionHandler` anywhere — service errors surface as default 500 responses with raw `RuntimeException` messages.
- Downstream services trust the `X-User-ID` header (no internal authentication); only the gateway validates JWTs.
- `ActivityRequest` has no Bean Validation annotations (user registration DTOs do).
- DTOs are duplicated between gateway and userservice; mapping is manual (no MapStruct/ModelMapper).
- The aiservice depends on live Gemini output; malformed AI responses fall back to a static default recommendation.

**Frontend**
- All URLs (gateway, Keycloak) are hardcoded — no `.env` configuration.
- `ActivityForm` expects prop `onActivityAdded`, but `App.jsx` passes `onActivitiesAdded`, and refresh relies on `window.location.reload()` — the list may not refresh after adding.
- The Redux `logout` action is never dispatched (OAuth `logOut()` is used instead), so `localStorage` retains stale `token`/`userId`.
- `GET /api/recommendations/user/{userId}` and the `/api/users/**` gateway route are currently unused by the SPA.
- Page `<title>` is still the default "Vite + React".

**Build environment**
- The poms declare Java 23 while many machines have 17 or a newer JDK — see [Building the Project](#building-the-project) for the exact flags used.

---
