# FAANG Social Network Platform

Welcome to the **FAANG Social Network**, a high-performance, microservices-based social platform built with Java 25, Spring Boot 4, Kafka, and Kubernetes.

This repository serves as the **Main Entry Point** for the entire ecosystem, orchestrating the deployment and integration of all sub-services.

---

## 🚀 Quick Start (Local Development)

The platform's shared infrastructure can be started with Docker Compose. Application services are built and run separately.

### Prerequisites
- Docker & Docker Compose
- Java 25 (optional, for local builds)
- Git

### Installation
```bash
# Clone the repository with all submodules
git clone --recursive https://github.com/bormoley1983/faang-main.git
cd faang-main

# Spin up the entire infrastructure (DBs, Brokers, Storage)
docker-compose up -d
```

---

## 🏗️ Architecture Overview

The system is composed of several specialized microservices communicating via **Kafka** for event-driven flows and **Redis** for real-time channels.

### Microservices
*   **[Account Service](./faang-account_service)**: User account management and balance.
*   **[User Service](./faang-user_service)**: User profiles, subscriptions, goals, skills, events, recommendations, and premium access.
*   **[Post Service](./faang-post_service)**: Text posts, image attachments (Minio), and hashtag indexing (Elasticsearch).
*   **[Hashtag Service](./faang-hashtag_service)**: Reserved standalone hashtag service. It is currently a buildable scaffold and is not deployed; Post Service still owns active hashtag indexing.
*   **[Notification Service](./faang-notification_service)**: Multi-channel notifications (Email, Telegram, SMS).
*   **[Analytics Service](./faang-analytics_service)**: Real-time event tracking and statistics.
*   **[Payment Service](./faang-payment_service)**: Transaction processing and currency exchange.
*   **[Project Service](./faang-project_service)**: Project collaboration and task management.
*   **[Achievement Service](./faang-achievement_service)**: Gamification and user rewards.
*   **[URL Shortener](./faang-url_shortener_service)**: Efficient URL management.

### Infrastructure (`faang-infra`)
Contains Kubernetes manifests, database initialization scripts, and CI/CD utility Dockerfiles.

---

## ☸️ Kubernetes & DevOps

The platform includes CI workflows and a GitOps-oriented delivery pipeline for the account service.

*   **CI**: Each service repository builds and tests pull requests and pushes to `dev-local`.
*   **Delivery**: The root `Jenkinsfile` publishes an immutable account-service image and updates its Kustomize tag.
*   **Deployment**: ArgoCD syncs manifests in `faang-infra/k8s`; Jenkins does not apply workloads directly.
*   **Observability**: Integrated Spring Boot Actuator with readiness/liveness probes.

---

## 🛠️ Tech Stack
- **Backend**: Java 25, Spring Boot 4.1.1, Spring Cloud (Feign, Gateway)
- **Database**: PostgreSQL 18 (Relational), Redis (Cache/Messaging)
- **Message Broker**: Apache Kafka
- **Search & Storage**: Elasticsearch 9.3 / Kibana 9.2.4, Minio (S3 Compatible)
- **Containerization**: Docker, Kubernetes (K8s)
- **DevOps**: Jenkins, ArgoCD

> **Note:** The root `docker-compose.yaml` launches infrastructure only (PostgreSQL, Redis, Kafka, Elasticsearch, Kibana, MinIO). Application services are built and run per-service via their Gradle wrappers or Dockerfiles. See each service's README for local run instructions.

### Hashtag Service status

Hashtag Service is registered as a submodule and has an independent CI build, but it does not yet expose an API or provide a deployable Spring Boot application. It must not be added to Compose or Kubernetes until its runtime contract, storage ownership, health checks, Docker image, and integration with Post Service are implemented.
