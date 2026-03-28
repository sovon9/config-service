# config-service

A robust, centralized Configuration Server built with Spring Cloud Config. This service manages external properties for applications across all environments, secured via Spring Security, backed by a local filesystem (Native), and integrated with Spring Cloud Bus + Kafka for real-time dynamic configuration updates.

---
## CURL to fetch config:
example: 
`` curl -u configUser:password \
http://localhost:8763/SpringCloud-Bus-Kafka/default/1.0.0 ``
---
## 1. Native Configuration Management

### What is it and why use it?
Typically, Spring Cloud Config uses a Git repository to store configuration files. However, by activating the `native` profile, we instruct the Config Server to load configuration files directly from the local filesystem. This is extremely useful for local development, offline testing, or environments where setting up a dedicated Git repo isn't strictly necessary.

### How to use it:
1. **Enable the profile:** In `application.properties`, we set `spring.profiles.active=native`.
2. **Define the path:** We specified `spring.cloud.config.server.native.searchLocations=file:///F:/configs/{application}/{profile}/{label}`.

**Understanding the Path Variables:**
- `{application}`: The name of the client microservice (e.g., `order-service`).
- `{profile}`: The active profile of the client (e.g., `dev`, `prod`).
- `{label}`: Usually represents a branch name in Git (e.g., `master`, `main`), but in native mode, it acts as another directory tier.

**Example Usage:**
If `order-service` requests its `dev` configuration with the `main` label, the Config Server will look for property files inside the directory:
`F:/configs/order-service/dev/main/`

---

## 2. Spring Security Integration

### Why do we need it?
Configuration files often contain highly sensitive data such as database passwords, API keys, and third-party secrets. If the Config Server is left unsecured, anyone (or any malicious script) that can access the network could download these secrets in plain text. 

### How it works:
We added `spring-boot-starter-security` and a `SecurityConfig` class to enforce HTTP Basic Authentication on all endpoints. 

In `application.properties`, we defined:
```properties
spring.security.user.name=configUser
spring.security.user.password=password
```

**Client Integration:** 
Any microservice that wants to fetch its configuration from this server *must* provide these credentials. In the client microservice's configuration, you will need to add:
```properties
spring.cloud.config.username=configUser
spring.cloud.config.password=password
```

---

## 3. Spring Cloud Bus & Kafka (Dynamic Refresh)

### The Problem it Solves:
Imagine you have 50 instances of an `inventory-service` running. If you change a database URL in the configuration file, you would normally have to restart all 50 instances, or manually hit the `/actuator/refresh` endpoint on *every single instance* for them to pick up the new changes. This is tedious and unscalable.

### What is Spring Cloud Bus?
Spring Cloud Bus links all your microservices together via a lightweight message broker (in our case, Apache Kafka). It is used to broadcast state changes (like configuration updates) across the entire distributed system.

### How to integrate and use it:

1. **Add Dependencies:** We included `spring-cloud-starter-bus-kafka`.
2. **Broker Configuration:** We pointed our app to the Kafka broker via `spring.kafka.bootstrap-servers=localhost:9092`. (Kafka and Zookeeper must be running locally on this port).
3. **Expose Actuator Endpoint:** We added `management.endpoints.web.exposure.include=busrefresh`.

### The Workflow (How to trigger an update):
1. You update a property in your local configuration file (e.g., inside `F:/configs/...`).
2. You send an empty `POST` request to the Config Server's bus refresh endpoint:
   `POST http://localhost:8763/actuator/busrefresh`
   *(Remember to pass your Basic Auth credentials!)*
3. The Config Server receives this request and publishes a "RefreshConfiguration" event to a specific Kafka topic.
4. **ALL** microservices connected to this Kafka broker automatically consume this event.
5. All microservices simultaneously reach back out to the Config Server, download their latest configurations, and hot-reload their `@RefreshScope` beans—without a single application restart!

---

## Quick Start / Running the Project

1. Ensure **Apache Kafka** is running on `localhost:9092`.
2. Ensure your directory structure matches your `searchLocations` (e.g., `F:/configs/...`).
3. Run `ConfigServiceApplication.java`.
4. Test fetching a config via browser or Postman: 
   `GET http://localhost:8763/{application}/{profile}/{label}`
   *(e.g., `http://localhost:8763/myapp/dev/main`)*
   **Credentials:** `configUser` / `password`

### Tech Stack
- **Spring Boot:** 3.5.7
- **Spring Cloud:** 2025.0.1
- **Java:** 17
- **Security:** HTTP Basic Auth
- **Message Broker:** Apache Kafka
