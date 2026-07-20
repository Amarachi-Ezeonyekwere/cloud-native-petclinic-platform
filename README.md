# Cloud-Native PetClinic Platform

> Production-grade cloud-native deployment of the Spring PetClinic 
> Microservices application on AWS, built as a full end-to-end 
> DevOps engineering portfolio project.

## What This Project Demonstrates

- Building and deploying a distributed microservices application 
  from source (not pre-built images)
- Production-realistic local development environment using Docker 
  Compose with MySQL persistence
- Externalized configuration management via Spring Cloud Config 
  with a dedicated config repository
- GitOps delivery via ArgoCD and Helm on Amazon EKS
- Infrastructure as Code with Terraform (VPC, EKS, RDS, ECR, IAM)
- End-to-end observability: Prometheus, Grafana, Zipkin

---

## Application Architecture

The platform consists of 7 Spring Boot microservices:

| Service | Responsibility | Port |
|---|---|---|
| Config Server | Centralized configuration for all services | 8888 |
| Discovery Server | Eureka service registry | 8761 |
| API Gateway | Single entry point, routes to downstream services | 8080 |
| Customers Service | Manages owners and pets data | 8081 |
| Visits Service | Manages pet visit records | 8082 |
| Vets Service | Manages veterinarian data | 8083 |
| Admin Server | Spring Boot Admin monitoring dashboard | 9090 |

---

## Repository Structure

This project follows a GitOps multi-repo pattern:

| Repository | Purpose |
|---|---|
| `cloud-native-petclinic-platform` (this repo) | Application source code and Dockerfiles |
| [petclinic-config](https://github.com/Amarachi-Ezeonyekwere/petclinic-config) | Externalized Spring Cloud Config files |
| [petclinic-infra](https://github.com/Amarachi-Ezeonyekwere/petclinic-infra) | Terraform infrastructure (AWS) |
| [petclinic-gitops](https://github.com/Amarachi-Ezeonyekwere/petclinic-gitops) | Helm charts and ArgoCD manifests |

---

## Local Development Setup

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Docker | Latest | [docs.docker.com](https://docs.docker.com/get-docker/) |
| Docker Compose | v2+ | Included with Docker Desktop |
| Java JDK | 17 | `sudo apt install openjdk-17-jdk` |

### Step 1 — Clone this repo

```bash
git clone https://github.com/Amarachi-Ezeonyekwere/cloud-native-petclinic-platform.git
cd cloud-native-petclinic-platform
```

### Step 2 — Clone the config repo alongside it

The config repo must live at `../petclinic-config` relative to this 
repo, because Docker Compose bind mounts it into the Config Server 
container from that path.

```bash
cd ..
git clone https://github.com/Amarachi-Ezeonyekwere/petclinic-config.git
cd cloud-native-petclinic-platform
```

Your folder structure should look like:
~/
├── cloud-native-petclinic-platform/   ← this repo
└── petclinic-config/                  ← config repo

### Step 3 — Set JAVA_HOME

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' >> ~/.bashrc
```

### Step 4 — Compile all services

This step compiles the Java source code and produces a JAR file 
for each service. Docker uses these JARs to build container images. 
This must be run before starting the stack.

```bash
./mvnw clean install -DskipTests
```

Expected output: `BUILD SUCCESS` across all 8 modules. Takes 
approximately 10 minutes on first run (dependency downloads).

### Step 5 — Start the full stack

```bash
docker compose up --build
```

Allow 3-5 minutes for all services to fully start and register 
with Eureka. Startup order is enforced via healthcheck dependencies:
Config Server → Discovery Server + MySQL + Zipkin → All other services.

### Step 6 — Verify everything is running

```bash
docker compose ps -a
```

All services except `genai-service` should show `Up`. 
`genai-service` requires an OpenAI API key (see below).

---

## Service URLs

| Service | URL | Credentials |
|---|---|---|
| PetClinic UI | http://localhost:8080 | None |
| Eureka Dashboard | http://localhost:8761 | None |
| Config Server | http://localhost:8888 | None |
| Admin Server | http://localhost:9090 | None |
| Zipkin Tracing | http://localhost:9411 | None |
| Grafana | http://localhost:3030 | admin / admin |
| Prometheus | http://localhost:9091 | None |

---

## GenAI Service (Optional)

The GenAI chatbot service requires an OpenAI API key. It is 
excluded from the default stack. To enable:

```bash
OPENAI_API_KEY=your_key_here docker compose --profile genai up
```

---

## Engineering Decisions

Key architectural decisions made in this project are documented 
in `docs/decisions/`. Highlights:

- **Build from source vs Docker Hub images** — All service images 
  are built from locally compiled Maven JARs, not pulled from Docker 
  Hub. This ensures artifact traceability and mirrors real CI/CD 
  pipeline behavior. See `docs/decisions/adr-001-build-vs-image.md`

- **MySQL over HSQLDB** — Production uses persistent MySQL (Amazon 
  RDS in AWS, containerized MySQL locally) rather than the in-memory 
  HSQLDB that ships for demo purposes. See 
  `docs/decisions/adr-002-mysql-over-hsqldb.md`

- **Native config profile** — Config Server reads from a local 
  filesystem bind mount in development, avoiding the need to push 
  config changes to GitHub to see them take effect locally. See 
  `docs/decisions/adr-003-native-config-profile.md`

---

## Production Deployment

Production infrastructure and deployment are documented in the 
companion repositories:

- **Infrastructure**: [petclinic-infra](https://github.com/Amarachi-Ezeonyekwere/petclinic-infra) — Terraform modules for VPC, EKS, RDS, ECR
- **GitOps**: [petclinic-gitops](https://github.com/Amarachi-Ezeonyekwere/petclinic-gitops) — Helm charts and ArgoCD application manifests

---

## License

Based on the [Spring PetClinic Microservices](https://github.com/spring-petclinic/spring-petclinic-microservices) 
open source project. See LICENSE for details.

