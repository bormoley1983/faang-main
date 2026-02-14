# FAANG Social Network Platform

Welcome to the **FAANG Social Network**, a high-performance, microservices-based social platform built with Java 21, Spring Boot 4, Kafka, and Kubernetes.

This repository serves as the **Main Entry Point** for the entire ecosystem, orchestrating the deployment and integration of all sub-services.

---

## 🚀 Quick Start (Local Development)

This platform is designed to be up and running with a single command using Docker Compose.

### Prerequisites
- Docker & Docker Compose
- Java 21 (optional, for local builds)
- Git

### Installation
```bash
# Clone the repository with all submodules
git clone --recursive https://github.com/your-username/faang-main.git
cd faang-main

# Spin up the entire infrastructure (DBs, Brokers, Storage)
docker-compose up -d
```

---

## 🏗️ Architecture Overview

The system is composed of several specialized microservices communicating via **Kafka** for event-driven flows and **Redis** for real-time channels.

### Microservices
*   **[Account Service](./faang-account_service)**: User account management and balance.
*   **[Post Service](./faang-post_service)**: Text posts, image attachments (Minio), and hashtag indexing (Elasticsearch).
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

The platform is production-ready with a full CI/CD pipeline and Kubernetes configurations.

*   **CI/CD**: Managed via `Jenkinsfile` in the root.
*   **Deployment**: ArgoCD/Rancher ready using manifests in `faang-infra/k8s`.
*   **Observability**: Integrated Spring Boot Actuator with readiness/liveness probes.

---

## 🛠️ Tech Stack
- **Backend**: Java 21, Spring Boot 4.0.2, Spring Cloud (Feign, Gateway)
- **Database**: PostgreSQL (Relational), Redis (Cache/Messaging)
- **Message Broker**: Apache Kafka
- **Search & Storage**: Elasticsearch 9.2, Minio (S3 Compatible)
- **Containerization**: Docker, Kubernetes (K8s)
- **DevOps**: Jenkins, ArgoCD
