# Cloud-Native PetClinic Platform

> A distributed Spring Boot microservices application, deployed to a
> production-shaped AWS/Kubernetes platform as a full end-to-end
> DevOps engineering portfolio project.

## What This Project Demonstrates

- Building and deploying a distributed microservices application from
  source (not pre-built images)
- A production-realistic local development environment via Docker
  Compose, with MySQL persistence
- Externalized configuration via Spring Cloud Config, reading from a
  dedicated config repository mounted at runtime
- Deployment to Amazon EKS via Helm, behind a single ALB entry point
- Dynamic node autoscaling via Karpenter
- Runtime credential sync from AWS Secrets Manager via External
  Secrets Operator — no database credentials ever stored in Kubernetes
  manifests or this repo
- Infrastructure as Code with Terraform (VPC, EKS, IAM/IRSA, RDS,
  ECR, Karpenter, observability) — 78 resources, 14 documented
  architecture decisions
- Log aggregation via Loki, running on EKS with S3-backed storage

## Live status

This application is deployed and running end-to-end on AWS — real
ALB, real EKS cluster, real RDS database. Deployment steps for the
full platform, across all four repositories, are documented in
[`petclinic-infra`'s GETTING-STARTED.md](https://github.com/Amarachi-Ezeonyekwere/petclinic-infra/blob/main/GETTING-STARTED.md).

Deployment is managed via ArgoCD, syncing automatically from
[`petclinic-gitops`](https://github.com/Amarachi-Ezeonyekwere/petclinic-gitops) —
any change pushed to that repo's `main` branch is detected and applied
automatically, with drift correction (`selfHeal`) if the cluster state
is ever manually changed outside of Git. See ADR-013 in `petclinic-infra`
for the ArgoCD configuration decisions.

## Application Architecture

7 Spring Boot microservices (an 8th, `genai-service`, exists in source
but is excluded from cloud deployment — no API key provisioned):

| Service | Responsibility | Port |
|---|---|---|
| Config Server | Centralized configuration for all services | 8888 |
| Discovery Server | Eureka service registry | 8761 |
| API Gateway | Single entry point, routes to downstream services | 8080 |
| Customers Service | Manages owners and pets data | 8081 |
| Visits Service | Manages pet visit records | 8082 |
| Vets Service | Manages veterinarian data | 8083 |
| Admin Server | Spring Boot Admin monitoring dashboard | 9090 |

## Observability

**Deployed to AWS (cloud environment):**
- **Zipkin** (`tracing-server`) — distributed request tracing across all services
- **Loki** — log aggregation, S3-backed storage, deployed via Terraform with its own IRSA role (see `petclinic-infra`)

**Local development only (`docker-compose.yml`), not part of the cloud deployment:**
- Prometheus — metrics scraping
- Grafana — metrics dashboards

## Repository Structure

This project follows a GitOps-oriented multi-repo pattern:

| Repository | Purpose |
|---|---|
| `cloud-native-petclinic-platform` (this repo) | Application source code and Dockerfiles |
| [petclinic-config](https://github.com/Amarachi-Ezeonyekwere/petclinic-config) | Externalized Spring Cloud Config files |
| [petclinic-infra](https://github.com/Amarachi-Ezeonyekwere/petclinic-infra) | Terraform infrastructure (AWS) — ADRs and troubleshooting journal live here |
| [petclinic-gitops](https://github.com/Amarachi-Ezeonyekwere/petclinic-gitops) | Helm charts, Ingress, Karpenter NodePools, External Secrets |

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

Folder structure should look like:
```
~/
├── cloud-native-petclinic-platform/   ← this repo
└── petclinic-config/                  ← config repo
```

### Step 3 — Set JAVA_HOME

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
echo 'export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64' >> ~/.bashrc
```

### Step 4 — Compile all services

```bash
./mvnw clean install -DskipTests
```
Expected output: `BUILD SUCCESS` across all 8 modules. ~10 minutes on
first run (dependency downloads).

### Step 5 — Start the full stack

```bash
docker compose up --build
```
Allow 3-5 minutes for all services to start and register with Eureka.
Startup order is enforced via healthcheck dependencies: Config Server
→ Discovery Server + MySQL + Zipkin → all other services.

### Step 6 — Verify

```bash
docker compose ps -a
```
All services except `genai-service` should show `Up`.

## Service URLs (local)

| Service | URL | Credentials |
|---|---|---|
| PetClinic UI | http://localhost:8080 | None |
| Eureka Dashboard | http://localhost:8761 | None |
| Config Server | http://localhost:8888 | None |
| Admin Server | http://localhost:9090 | None |
| Zipkin Tracing | http://localhost:9411 | None |
| Grafana | http://localhost:3030 | admin / admin |
| Prometheus | http://localhost:9091 | None |

## GenAI Service (Optional, local only)

Requires an OpenAI API key. Excluded from both the default local
stack and the cloud deployment.
```bash
OPENAI_API_KEY=your_key_here docker compose --profile genai up
```

## Deploying to AWS

Full multi-repo deployment steps — infrastructure, Kubernetes
manifests, config mounting, image builds — are documented in
[`petclinic-infra`'s GETTING-STARTED.md](https://github.com/Amarachi-Ezeonyekwere/petclinic-infra/blob/main/GETTING-STARTED.md).

Every real issue hit building the cloud deployment (Spring profile
bugs, Kubernetes-specific config gaps, IAM permission gaps, image
architecture mismatches) is documented with root cause and fix in
[`petclinic-infra`'s TROUBLESHOOTING.md](https://github.com/Amarachi-Ezeonyekwere/petclinic-infra/blob/main/docs/TROUBLESHOOTING.md).

## Architecture decisions

All 14 architecture decision records for this platform — infrastructure
design, cost trade-offs, security scoping, and the account-level
constraints worked around while building this — live in
[`petclinic-infra/docs/adr/`](https://github.com/Amarachi-Ezeonyekwere/petclinic-infra/tree/main/docs/adr),
numbered in the order they were actually made.

---

*This project's application code originates from the
[Spring PetClinic Microservices](https://github.com/spring-petclinic/spring-petclinic-microservices)
open source sample. The infrastructure, Kubernetes deployment,
observability, and GitOps configuration built on top of it are
original work, see the repositories above for the full build.*