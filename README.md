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

---

## Project Status

| Component | Status |
|---|---|
| user-service | ✅ Complete |
| booking-service | ✅ Complete |
| payment-service | ✅ Complete |
| notification-service | ✅ Complete |
| analytics-service | 🔄 In progress |
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
