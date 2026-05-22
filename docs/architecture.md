# Architecture Overview

This document expands on the high-level architecture described in `README.md`. It is focused on conceptual design only; it does not represent a full implementation.

---

## 1. System Overview

The platform is an internal web application used by internal users to query and visualise data from multiple data sources (cloud and on-premise).

Core services:

| Service | Name          | Description |
|--------|---------------|-------------|
| Frontend | `atlas-portal` | Web UI for internal users |
| Backend API | `orion-api` | Business logic and data aggregation API |
| Workflow orchestration | Apache Airflow | Batch and scheduled workflows / ETL |
| Database | PostgreSQL | Persistent storage for application and pipeline state |
| External data sources | N/A | Cloud and on-premise systems queried by the platform |

The platform is designed to run on Kubernetes, with one namespace per environment: dev, staging, and prod.

---

## 2. Component Responsibilities

### 2.1 `atlas-portal` (Web Frontend)

- Serves a single-page application (SPA) over HTTPS.
- Communicates only with `orion-api`; it does not access databases or external data sources directly.
- Handles user interactions, filtering, and presentation logic.
- Uses environment-specific API base URLs, e.g.:
  - `[dev.sornsub.online](https://dev.sornsub.online/api)`
  - `[staging.sornsub.online](https://staging.sornsub.online/api)`
  - `[prod.sornsub.online](https://prod.sornsub.online/api)`

### 2.2 `orion-api` (Backend API)

- Exposes REST endpoints for `atlas-portal` and internal tools.
- Implements business logic, data aggregation, and access control.
- Reads and writes application data in PostgreSQL.
- Performs:
  - Real-time queries to external cloud APIs where low latency is required.
  - Reads of pre-computed or batch-loaded data written by Airflow.
- Enforces authentication and authorisation for all client requests.

### 2.3 Apache Airflow (Workflow Orchestration)

- Runs scheduled DAGs for:
  - Extracting data from external sources.
  - Transforming and cleaning the data.
  - Loading results into PostgreSQL for `orion-api` to consume.
- Uses PostgreSQL as its metadata store (separate schema or database).
- Connects to external systems using managed connections and credentials sourced from the secret management solution.

### 2.4 PostgreSQL

- Relational database for:
  - Application data used by `orion-api`.
  - Airflow metadata and job state.
- Deployed as a managed database service or stateful workload with:
  - Encrypted storage.
  - Network access restricted to platform services.
- Not directly exposed to users or external networks.

### 2.5 External Cloud and On-Premise Data Sources

Representative examples (illustrative only):

- Cloud APIs: analytics services, SaaS platforms, or internal cloud microservices.
- On-premise systems: legacy databases or line-of-business applications reachable via VPN or private link.
- File/object storage: S3-compatible or similar services used by Airflow for intermediate artefacts.

Connections to these systems are secured and configured through secrets and controlled network paths.

---

## 3. Traffic Flow

### 3.1 High-Level Request Flow

1. **User** accesses the platform via browser:
   - `[dev.sornsub.online](https://dev.sornsub.online)` (dev)
   - `[staging.sornsub.online](https://staging.sornsub.online)` (staging)
   - `[prod.sornsub.online](https://prod.sornsub.online)` (prod)
2. **Ingress controller** receives HTTPS traffic and performs TLS termination.
3. Ingress routing sends:
   - `/` and static assets → `atlas-portal`
   - `/api/*` → `orion-api`
4. `atlas-portal` calls `orion-api` for data and actions.
5. `orion-api`:
   - Queries PostgreSQL for stored data.
   - Optionally calls external data sources for additional or real-time data.
6. Responses flow back through the ingress to the user’s browser.

### 3.2 Batch / Data Pipeline Flow

1. Airflow DAG is scheduled or triggered.
2. Airflow tasks connect to external cloud or on-premise systems.
3. Data is extracted, transformed, and loaded into PostgreSQL.
4. `orion-api` reads the processed results and exposes them to `atlas-portal`.

---

## 4. Environment Separation

Each environment is isolated using both DNS and Kubernetes namespaces.

| Environment | Domain                | Namespace         | Purpose |
|------------|-----------------------|-------------------|---------|
| Dev        | `dev.sornsub.online`     | `platform-dev`     | Active development and integration testing |
| Staging    | `staging.sornsub.online` | `platform-staging` | Pre-production validation |
| Prod       | `prod.sornsub.online`    | `platform-prod`    | Live internal usage |

Key separation aspects:

- Independent deployments and configuration per environment.
- Distinct secrets and credentials per environment.
- Dev and staging use synthetic or anonymised data; prod uses live internal data.
- Network policies can differ by environment (e.g. more relaxed in dev, stricter in prod).

---

## 5. High Availability and Scalability Assumptions

The design assumes that:

- **Stateless services** (`atlas-portal`, `orion-api`, Airflow web/scheduler/workers) run multiple replicas behind Kubernetes Services.
- **Ingress** is backed by multiple ingress controller pods, with load balancing from the underlying platform.
- **PostgreSQL** uses a managed HA configuration (multi-zone replicas) or an equivalent highly available setup.
- **Horizontal scaling**:
  - Frontend and API scale out by adding replicas.
  - Airflow workers scale based on workload.
- **Vertical scaling** is available by adjusting resource limits/requests for Pods and database tiers.

Typical targets (illustrative):

| Component      | HA Strategy                 | Scaling Approach              |
|---------------|-----------------------------|-------------------------------|
| `atlas-portal` | ≥2 replicas in staging/prod | Horizontal Pod Autoscaler     |
| `orion-api`   | ≥2 replicas in staging/prod | Horizontal Pod Autoscaler     |
| Airflow       | Multiple workers            | Worker pool scaling           |
| PostgreSQL    | Managed failover / replicas | Instance sizing + read replicas (if used) |

---

## 6. Network Security Boundaries

The design uses layered boundaries to control access:

1. **Public entry point**
   - Only the ingress controller is exposed externally.
   - All access uses HTTPS with certificates managed by cert-manager.
   - No direct access to `orion-api`, Airflow, or PostgreSQL from the public internet.

2. **Application layer (cluster internal)**
   - `atlas-portal`, `orion-api`, and Airflow components communicate via Kubernetes Services.
   - Kubernetes NetworkPolicies restrict:
     - Access to PostgreSQL to only authorised services.
     - Access to Airflow web UI to internal/admin users.

3. **Data layer**
   - PostgreSQL is not internet-accessible.
   - Connections require valid credentials from the secret store and originate from allowed Pods.

4. **External systems**
   - Cloud APIs accessed over TLS; egress may be routed through NAT with a controlled IP.
   - On-premise systems accessed over VPN or private link with firewall rules limiting ports and sources.

5. **Environment isolation**
   - No cross-namespace service access for core application paths.
   - Separate secret paths for dev, staging, and prod.

This model provides clear boundaries and helps minimise blast radius for misconfigurations or incidents.
