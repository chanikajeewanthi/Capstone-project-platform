# Capstone Project - Platform

Core infrastructure services for the Capstone Project academic management system. This repository contains the Service Registry, Config Server, and API Gateway — the shared backbone that the business services (`student-service`, `program-service`, `enrollment-service`) depend on.

## Components

| Component | Description | Port |
|---|---|---|
| `service-registry` | Eureka server — service discovery for all microservices | 9001 |
| `config-server` | Centralized configuration server for all services | 9000 |
| `api-gateway` | Single entry point routing external requests to internal services | 7000 |

## Architecture

- **Service Registry** must be the first component to start. All other services register with it and discover each other through it.
- **Config Server** loads configuration from local files under `config-server/src/main/resources/configurations/`, using the `native` Spring Cloud Config profile (no external Git backend). It serves configuration to every business service on startup.
- **API Gateway** routes external traffic to the registered services (e.g. `enrollment-service`) using Eureka for service discovery.

Business services (in the separate `Capstone-project-services` repository) depend on this repository being up and reachable before they can start successfully.

## Prerequisites

- Java 25 (or the version specified in each module's `pom.xml`)
- Maven 3.9+
- Node.js + PM2 (`npm install -g pm2`)
- Network access between this VM and the VM(s) running the business services (internal DNS name `config.platform` must resolve correctly)

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/<your-username>/Capstone-project-platform.git
cd Capstone-project-platform
```

### 2. Pull latest changes

```bash
git pull
```

### 3. Build all modules

```bash
mvn clean package -DskipTests
```

This produces a runnable `.jar` for each module under its own `target/` folder.

### 4. Start all components with PM2

```bash
pm2 start ecosystem.config.js
pm2 status
```

Start order matters conceptually (Service Registry → Config Server → API Gateway), but PM2 starts them concurrently by default — if a component fails to register on first boot, restart it once the others are confirmed online:

```bash
pm2 restart service-registry
pm2 restart config-server
pm2 restart api-gateway
```

### 5. Verify everything is running

**Eureka dashboard:**
```
http://<external-ip>:9001
```

You should see `API-GATEWAY` listed as `UP`. Business services will also appear here once they start successfully and can reach this platform.

**Config Server health check:**
```bash
curl http://localhost:9000/actuator/health
```

**Fetch a specific service's config (sanity check):**
```bash
curl http://localhost:9000/student-service/default
```

## Managing Configuration for Business Services

All database URLs, credentials, and per-service settings for `student-service`, `program-service`, and `enrollment-service` live here, **not** in their own repositories:

```
config-server/src/main/resources/configurations/services/
├── student-service.yaml
├── program-service.yaml
└── enrollment-service.yaml
```

Shared/global settings (Eureka URL, logging levels, etc.) live in:

```
config-server/src/main/resources/configurations/application.yaml
```

**Any time you edit a config file here, you must rebuild and restart the Config Server for the change to take effect:**

```bash
mvn clean package -pl config-server -DskipTests
pm2 restart config-server
```

Restarting the Config Server does **not** automatically refresh already-running business services — restart those separately (on their own VM) after the Config Server is back up:

```bash
pm2 restart student-service enrollment-service program-service
```

## Common Issues

| Symptom | Likely Cause | Fix |
|---|---|---|
| Business services can't fetch config on startup | Config Server not running, or `config.platform` DNS not resolving | `curl http://config.platform:9000/actuator/health` from the services VM |
| Config change doesn't take effect | Config Server jar wasn't rebuilt after editing YAML | Rebuild with `mvn clean package -pl config-server` and restart |
| Only `API-GATEWAY` shows in Eureka, no business services | Business services are crash-looping (usually a DB connection issue) — not a platform-side problem | Check `pm2 logs` on the services VM |
| `mvn clean package` fails with "Child module ... does not exist" | Submodule/module folder is empty | Run `git submodule update --init --recursive` if applicable, or re-clone |

## Repository Structure

```
Capstone-project-platform/
├── service-registry/
├── config-server/
│   └── src/main/resources/configurations/
│       ├── application.yaml
│       ├── platform/
│       │   ├── service-registry.yaml
│       │   └── api-gateway.yaml
│       └── services/
│           ├── student-service.yaml
│           ├── program-service.yaml
│           └── enrollment-service.yaml
├── api-gateway/
├── ecosystem.config.js    (PM2 process definitions)
├── pom.xml                (parent POM, aggregates all modules)
└── README.md
```
