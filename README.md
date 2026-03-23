# Taxi Booking Platform

A cloud-native, event-driven taxi booking platform built with a microservices architecture.  
Designed for high availability, real-time responsiveness, and production-grade observability.

> **Note:** This repository contains architecture documentation only. The source code is maintained in a private repository.

---

## Architecture Overview

The platform is organized as a set of independent microservices, each owning its domain and communicating asynchronously via Apache Kafka.

```
┌─────────────────────────────────────────────────────────────┐
│                        Clients                              │
│              Passenger App    Driver App                    │
│              (React Native)  (React Native)                 │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTPS
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   API Gateway                               │
│           Spring Cloud Gateway + Spring Security            │
│           JWT authentication / rate limiting                │
└──────┬──────────────┬──────────────┬────────────────────────┘
       │              │              │
       ▼              ▼              ▼
┌──────────┐  ┌──────────────┐  ┌──────────────┐
│   User   │  │   Booking    │  │   Payment    │
│ Service  │  │   Service    │  │   Service    │
└──────────┘  └──────────────┘  └──────────────┘
       │              │              │
       └──────────────┼──────────────┘
                      │ Apache Kafka
                      ▼
        ┌─────────────────────────────┐
        │  notification-service       │
        │  analytics-service          │
        └─────────────────────────────┘
```

### Kafka Event Topics

| Topic | Producer | Consumers |
|---|---|---|
| `user-registered` | user-service | notification-service |
| `booking-events` | booking-service | notification-service, analytics-service |
| `payment-events` | payment-service | notification-service, analytics-service |

---

## Services

### user-service
Handles user registration, authentication, and profile management.  
Produces `user-registered` events on Kafka.  
Secrets and credentials injected via Kubernetes Secrets.

### booking-service
Manages the full booking lifecycle: request, matching, confirmation, and cancellation.  
Publishes `booking-events` to Kafka for downstream consumption.

### payment-service
Processes payments and publishes `payment-events`.  
Isolated from other services to limit blast radius in case of failure.

### notification-service
Consumes Kafka events and dispatches real-time push notifications to passengers and drivers.

### analytics-service
Consumes Kafka events and aggregates metrics for operational dashboards.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Java 17, Spring Boot, Spring Cloud Gateway, Spring Security |
| Messaging | Apache Kafka |
| Database | PostgreSQL (per service), Flyway migrations |
| Cache | Redis |
| Containerization | Docker, Docker Compose (local), Kubernetes / AKS (cloud) |
| CI/CD | GitHub Actions |
| Observability | Prometheus, Grafana, OpenTelemetry Collector, Spring Actuator |
| Testing | JUnit 5, Spring Integration Tests, Postman / Newman, k6 |
| Mobile | React Native (Passenger app, Driver app) |

---

## Infrastructure

### Local Development

All services are orchestrated locally via Docker Compose:

```
docker-compose up --build
```

Services, Kafka, Zookeeper, PostgreSQL, and Redis start together.  
API Gateway is exposed on `http://localhost:8080`.

### Cloud Deployment — Azure Kubernetes Service (AKS)

Production deployment targets AKS.  
Each microservice has its own Kubernetes Deployment, Service, and ConfigMap.  
Sensitive configuration (database credentials, JWT secrets) is injected via Kubernetes Secrets — never committed to the repository.

```
kubectl apply -f devops/k8s/
```

---

## Observability

| Tool | Role |
|---|---|
| Prometheus | Scrapes `/actuator/prometheus` on each service |
| Grafana | Operational dashboards (latency, throughput, error rates) |
| OpenTelemetry Collector | Distributed tracing across services |
| Spring Actuator | Health checks at `/actuator/health` |

---

## CI/CD Pipeline

GitHub Actions pipeline runs on every push:

1. Unit tests (JUnit 5)
2. Spring integration tests
3. API test suite (Postman / Newman)
4. Load tests (k6 smoke)
5. Docker image build
6. Deployment to AKS (on `main` branch)

---

## Mobile Applications

Two React Native applications are developed in parallel with the backend:

- **Passenger App** — booking flow, real-time tracking, payment
- **Driver App** — trip acceptance, navigation, earnings

Both consume the API Gateway over HTTPS and receive real-time updates via push notifications.

# Taxi Booking Platform

A cloud-native, event-driven taxi booking platform built with a microservices architecture.  
Designed for high availability, real-time responsiveness, and production-grade observability.

> **Note:** This repository contains architecture documentation only. The source code is maintained in a private repository.

---

## Architecture Overview

The platform is organized as 12 independent microservices communicating asynchronously via Apache Kafka, exposed through a single API Gateway, and discoverable via Eureka.

```
┌──────────────────────────────────────────────────────────────┐
│                          Clients                             │
│               Passenger App       Driver App                 │
│               (React Native)     (React Native)              │
└───────────────────────┬──────────────────────────────────────┘
                        │ HTTPS / JWT
                        ▼
┌──────────────────────────────────────────────────────────────┐
│                      API Gateway                             │
│            Spring Cloud Gateway — port 8080                  │
│            JWT validation · rate limiting · routing          │
└───┬──────────┬────────┬────────┬──────────┬──────────────────┘
    │          │        │        │          │
    ▼          ▼        ▼        ▼          ▼
┌────────┐ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐
│  auth  │ │ user │ │driver│ │book- │ │ payment  │
│service │ │  svc │ │  svc │ │ ing  │ │  service │
└────────┘ └──────┘ └──────┘ └──────┘ └──────────┘
    │          │        │        │          │
    └──────────┴────────┴────────┴──────────┘
                        │ Apache Kafka
           ┌────────────┼──────────────────────┐
           ▼            ▼                      ▼
    ┌────────────┐ ┌──────────┐        ┌──────────────┐
    │notification│ │analytics │        │ geolocation  │
    │  service   │ │ service  │        │   service    │
    └────────────┘ └──────────┘        └──────────────┘
           │
           ▼
    ┌─────────────┐   ┌──────────────┐
    │chat-service │   │review-service│
    └─────────────┘   └──────────────┘

              ┌──────────────────┐
              │ service-discovery│
              │  Eureka — 8761   │
              └──────────────────┘
```

---

## Services

| Service | Responsibility |
|---|---|
| **auth-service** | JWT issuance and token validation. Tokens expire after 15 min. |
| **user-service** | Passenger account management and profile. |
| **driver-service** | Driver account, status, and availability management. |
| **booking-service** | Full ride lifecycle: request, matching, confirmation, cancellation. |
| **geolocation-service** | Real-time driver location tracking. |
| **payment-service** | Payment processing (production stub, production-ready interface). |
| **notification-service** | Consumes Kafka events and dispatches push notifications. |
| **chat-service** | In-trip messaging between passenger and driver. |
| **review-service** | Post-ride ratings and reviews. |
| **analytics-service** | Consumes Kafka events and aggregates operational metrics. |
| **api-gateway** | Single entry point — routing, JWT validation, rate limiting (Spring Cloud Gateway). |
| **service-discovery** | Eureka server for service registration and discovery. |
| **shared-libs** | Shared DTOs, utilities, error response structure, and common config. |

### Kafka Event Topics

| Topic | Producer | Consumers |
|---|---|---|
| `user-registered` | user-service | notification-service |
| `booking-events` | booking-service | notification-service, analytics-service |
| `payment-events` | payment-service | notification-service, analytics-service |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Java 17, Spring Boot, Spring Cloud Gateway, Spring Security |
| Architecture | Maven multi-module monorepo |
| Messaging | Apache Kafka + Zookeeper |
| Database | PostgreSQL (per service, `taxi-net` network, persistent volume) |
| Cache | Redis |
| Service discovery | Eureka |
| Containerization | Docker, Docker Compose (local), Kubernetes / AKS (cloud) |
| CI/CD | GitHub Actions |
| Observability | Prometheus, Grafana, OpenTelemetry Collector (`otel-collector-config.yml`) |
| Logging | Logback JSON — console + rolling file (10MB / 30 days) |
| Testing | JUnit 5, Spring Integration Tests, Postman / Newman, k6 |
| Mobile | React Native (Passenger app, Driver app) |

---

## Infrastructure

### Local Development

```bash
# Build all Maven modules
mvn clean package

# Start the full stack (all services + Kafka + Zookeeper + PostgreSQL)
docker compose up --build
```

- Eureka dashboard: `http://localhost:8761`
- API Gateway: `http://localhost:8080`
- Database state persisted in the `postgres-data` volume

```bash
# Stop the stack
docker compose down

# Stop and remove volumes
docker compose down -v
```

### Cloud Deployment — Azure Kubernetes Service (AKS)

Production deployment targets AKS.  
Each microservice has its own Kubernetes Deployment, Service, and ConfigMap.  
All secrets (database credentials, JWT signing key) are injected via Kubernetes Secrets — never committed to the repository.

```bash
kubectl apply -f devops/k8s/
```

---

## Observability

| Tool | Role |
|---|---|
| Prometheus | Scrapes `/actuator/prometheus` on each service |
| Grafana | Operational dashboards — latency, throughput, error rates |
| OpenTelemetry Collector | Distributed tracing across all services |
| Spring Actuator | Health checks at `/actuator/health` |

---

## Logging

All services share a common `logback-spring.xml` configuration:

- JSON-formatted logs on console and rolling file (`logs/app.log`)
- Daily rotation, max 10MB per file, 30 days retention
- Default level: `INFO` in production, `DEBUG` in development

---

## Error Handling

All services return errors in a consistent JSON structure:

```json
{
  "code": 404,
  "message": "Booking not found",
  "timestamp": "2025-06-01T10:30:00Z"
}
```

---

## Security

- JWT tokens issued by **auth-service**, validated at the API Gateway
- Token expiry: 15 minutes — re-authentication required
- JWT signing key configured via `jwt.secret` in `application.yml` (K8s Secret in production)
- All sensitive configuration injected via environment variables or Kubernetes Secrets
- No credentials committed to any repository
- Input validation enforced at gateway and service level

---

## CI/CD Pipeline

GitHub Actions pipeline runs on every push:

1. Unit tests (JUnit 5)
2. Spring integration tests
3. API test suite (Postman / Newman)
4. Load tests (k6 smoke)
5. Docker image build per service
6. Deployment to AKS (`main` branch only)

---

## Mobile Applications

Two React Native applications in active development:

- **Passenger App** — booking flow, real-time tracking, in-trip chat, payment, post-ride review
- **Driver App** — trip acceptance, real-time navigation, earnings, availability toggle

Both communicate with the API Gateway over HTTPS and receive real-time updates via push notifications.

---

## Project Status

| Component | Status |
|---|---|
| auth-service | ✅ Complete |
| user-service | ✅ Complete |
| driver-service | ✅ Complete |
| booking-service | ✅ Complete |
| geolocation-service | ✅ Complete |
| notification-service | ✅ Complete |
| payment-service | ✅ Complete |
| chat-service | ✅ Complete |
| review-service | ✅ Complete |
| analytics-service | ✅ Complete |
| api-gateway | ✅ Complete |
| service-discovery | ✅ Complete |
| Passenger App (React Native) | 🔄 In progress |
| Driver App (React Native) | 🔄 In progress |

---

## Security

- All inter-service communication is authenticated via JWT tokens validated at the API Gateway
- Kubernetes Secrets for all sensitive configuration
- No credentials or secrets are committed to any repository
- Input validation enforced at API Gateway and service level

---

## License

Private project — all rights reserved.
