# Operations and Runbook Overview

This document describes operational practices for the platform: monitoring, logging, health checks, alerting, incident response, backup, and related topics. It is design-focused and does not imply a fully implemented operations stack.

---

## 1. Monitoring Approach

Monitoring focuses on service health, performance, and capacity across all environments.

### 1.1 Metrics Collection

Assumed tooling:

- **Prometheus** for metrics collection.
- **Grafana** for dashboards and visualisation.

Key metrics:

| Area          | Examples |
|---------------|----------|
| HTTP services | Request rate, error rate (4xx/5xx), latency (p50/p95/p99) for `atlas-portal` and `orion-api` |
| Workflows     | Airflow DAG/task success/failure counts and durations |
| Database      | Connection count, query latency, cache hit ratio, disk usage |
| Kubernetes    | Pod CPU/memory, restarts, node health, resource saturation |

### 1.2 Dashboards

Suggested dashboards:

- **Application Overview**: traffic, errors, and latency per environment.
- **Airflow**: DAG run outcomes, queued vs. running tasks.
- **Database**: slow queries, connection pool utilisation, storage headroom.
- **Infrastructure**: node health, capacity utilisation, ingress traffic.

---

## 2. Logging Approach

Logging is structured and centralised.

### 2.1 Log Generation

- All services log to `stdout`/`stderr` in structured JSON.
- Log fields include:
  - `timestamp`
  - `level`
  - `service`
  - `environment`
  - `version` (image tag)
  - `trace_id` / `request_id`
  - Message and context fields

### 2.2 Log Collection and Storage

Assumed pattern:

- **Fluent Bit** (or similar) runs as a DaemonSet on cluster nodes.
- Logs are shipped to a central log store such as:
  - Loki
  - Elasticsearch/OpenSearch
  - Cloud-native logging service

Retention (illustrative):

| Environment | Suggested Retention |
|------------|---------------------|
| Dev        | ~14 days            |
| Staging    | ~30 days            |
| Prod       | ~90 days            |

Access is role-based, with production logs restricted to authorised staff.

---

## 3. Health Checks

Each service exposes a small set of HTTP endpoints used by Kubernetes and monitoring tools.

| Endpoint        | Purpose                                    | Consumer                 |
|----------------|--------------------------------------------|--------------------------|
| `/health/live` | Liveness: process running, basic checks    | Kubernetes `livenessProbe` |
| `/health/ready`| Readiness: ready to serve traffic          | Kubernetes `readinessProbe` |
| `/metrics`     | Prometheus metrics in text format          | Prometheus               |

Notes:

- A failing **liveness** check restarts the container.
- A failing **readiness** check removes the pod from service endpoints but does not restart it immediately.
- Airflow can expose its own health endpoint; it should be integrated into probes and dashboards.

---

## 4. Alerting

Alerting is based on Prometheus alert rules and Alertmanager routing.

### 4.1 Core Alerts

| Category       | Example Condition                                          | Severity  |
|----------------|------------------------------------------------------------|-----------|
| Availability   | High 5xx error rate for `orion-api` for > 2 minutes        | Critical  |
| Performance    | p99 latency > defined threshold for sustained period       | Warning   |
| Workflows      | Airflow DAG consecutive failures or missed scheduled runs  | Warning   |
| Infrastructure | Node or pod CPU/memory pressure > threshold                | Warning   |
| Database       | Connection saturation or high query latency                | Critical  |
| Certificates   | TLS certificates expiring within 14 days                   | Warning   |

### 4.2 Alert Routing

Typical channels:

- Production-critical alerts → paging system and a high-priority channel (e.g. `#alerts-prod`).
- Non-production alerts → lower-priority channels (e.g. `#alerts-dev`, `#alerts-staging`).
- Infrastructure and certificate alerts → separate infra channel (e.g. `#alerts-infra`).

Alert policies can differ per environment to reduce noise in dev while maintaining strong coverage in prod.

---

## 5. Incident Response Flow

A lightweight incident process ensures structured handling of production issues.

### 5.1 High-Level Flow

1. **Detection**
   - Alert fires or an issue is reported by users.
2. **Triage**
   - On-call engineer verifies impact and scope using dashboards and logs.
3. **Containment**
   - Options include:
     - Rolling back to a previous release.
     - Temporarily scaling services up or down.
     - Disabling problematic features via configuration/flags.
4. **Resolution**
   - Root cause is identified.
   - Fix is implemented, reviewed, and deployed through the normal release pipeline.
5. **Post-Incident Review**
   - A short document records:
     - Timeline.
     - Root cause and contributing factors.
     - What worked, what did not.
     - Follow-up actions (e.g. improved tests, new alerts).

The level of formality may vary by severity, but production incidents that affect users should always be reviewed.

---

## 6. Backup and Recovery Considerations

Backups primarily concern PostgreSQL and critical configuration.

### 6.1 PostgreSQL

- Use managed database backups or scheduled logical/physical backups.
- Store backups in a secure, durable location with appropriate retention.
- Periodically test restore procedures in non-production environments.

Illustrative retention:

| Environment | Backup Frequency | Retention     |
|------------|------------------|---------------|
| Dev        | Daily snapshot   | Short term    |
| Staging    | Daily snapshot   | Medium term   |
| Prod       | Hourly or daily  | Longer term   |

### 6.2 Configuration and Secrets

- Configuration is version-controlled; Helm values and IaC definitions serve as “configuration backups”.
- Secrets are stored in a secure secret manager (e.g. Vault) that supports its own backup mechanisms.

Recovery goals should be defined in terms of:

- **RPO** (Recovery Point Objective): maximum acceptable data loss window.
- **RTO** (Recovery Time Objective): target time to restore service.

---

## 7. Secret Management

Secret handling follows the principles described in `README.md`.

### 7.1 Storage

- Secrets (e.g. database passwords, API keys, OAuth credentials) are stored in:
  - HashiCorp Vault, or
  - An equivalent managed secret service.
- Secrets are grouped by environment, e.g.:

  ```text
  secret/platform/dev/orion-api/...
  secret/platform/staging/orion-api/...
  secret/platform/prod/orion-api/...
