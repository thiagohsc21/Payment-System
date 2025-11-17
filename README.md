# Payment Orchestrator

![Status](https://img.shields.io/badge/status-in%20development-yellow)
![Language](https://img.shields.io/badge/language-Java%2017-orange)
![Framework](https://img.shields.io/badge/framework-Spring%20Boot%203-green)
![License](https://img.shields.io/badge/license-MIT-blue)

## About the Project

This project is a robust, distributed payment orchestration platform designed to simulate enterprise-grade financial transaction processing. Developed in **Java 17** with **Spring Boot**, it focuses on high availability, data consistency, and asynchronous processing patterns.

The goal is to demonstrate advanced software engineering concepts such as **idempotency**, **event-driven architecture**, and **cloud-native deployment** (AWS/Kubernetes), moving beyond simple CRUD applications to solve real-world distributed system challenges.

## Architecture Overview

The system follows a **Microservices-ready** architecture, decoupling the reception of requests from their processing:

1.  **Payment API (Synchronous):** A RESTful API acts as the entry point, receiving transaction requests, performing initial validations, and persisting the initial state (PENDING) in a **PostgreSQL** database.
2.  **Message Broker:** Validated requests are published as events to an **Apache Kafka** topic, ensuring non-blocking user experience and system resilience.
3.  **Payment Worker (Asynchronous):** A dedicated consumer service reads events from Kafka, integrates with external payment gateways (e.g., Stripe), and updates the transaction status.
4.  **Audit & Logging:** All transaction steps and raw payloads are asynchronously logged into **MongoDB** for traceability and auditing purposes.

## Tech Stack

* **Core:** Java 17, Spring Boot 3 (Web, Data JPA, Data Mongo).
* **Messaging:** Apache Kafka.
* **Databases:** PostgreSQL (Relational/Transactional), MongoDB (NoSQL/Logs).
* **DevOps & Cloud:** Docker, Kubernetes (EKS), AWS, CI/CD (GitHub Actions).
* **Testing:** JUnit 5, Mockito, Testcontainers.

## Project Documentation

Detailed design and architectural decisions can be found in the documentation folder:

* **[System Requirements](documentation/requirements.md):** Functional requirements (Payment processing, Status query) and Non-Functional constraints (Idempotency, Consistency).
* **[System Architecture](documentation/architecture.md):** C4 Model diagrams, component breakdown, and data flow description.
* **[API Reference](documentation/api.md):** JSON contracts for the REST endpoints.

