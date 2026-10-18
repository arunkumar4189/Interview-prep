# Cloud-native deployment, CI/CD, and observability (detailed)

Mapped to your stack: **AWS (EKS, EC2, S3), Docker, Jenkins, GitLab CI, Grafana, New Relic**, plus edge **Akamai / APIGEE / 42Crunch**.

---

## 1. Container image

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/campaign-api.jar app.jar
USER nonroot
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

**Practices:**

- One process per container.
- Config via env (`SPRING_DATASOURCE_URL`), secrets from K8s Secret / AWS Secrets Manager—not baked into image.
- Tag image with **git SHA**, not `latest` only.

---

## 2. Kubernetes (EKS) deployment sketch

### API deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: campaign-api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    spec:
      containers:
        - name: api
          image: 123456789.dkr.ecr.ap-south-1.amazonaws.com/campaign-api:abc123
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              memory: "2Gi"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            periodSeconds: 30
```

### Worker deployment

- **No Ingress**—workers only consume Kafka/RabbitMQ.
- **Graceful shutdown:** on `SIGTERM`, stop polling, finish current chunk, commit offset, exit (30–60s `terminationGracePeriodSeconds`).

### HPA

- API: CPU or request rate.
- Workers: **custom metric** `kafka_consumer_lag` or queue depth—not CPU alone.

### PDB

```yaml
minAvailable: 1
```

So node drain doesn’t take all API pods.

---

## 3. Probes — incident story to tell

**Bad:** Liveness probe calls Postgres. DB blip → all pods restart → thundering herd → outage.

**Good:**

- **Readiness:** can reach DB + broker (remove from Service endpoints if not).
- **Liveness:** JVM responsive / thread pool not deadlocked—**not** full dependency chain.

---

## 4. CI/CD pipeline (GitLab CI or Jenkins)

```text
stage: build
  - mvn -B verify (unit + integration with Testcontainers)

stage: image
  - docker build & push to ECR

stage: deploy-staging
  - helm upgrade --install campaign-api ./chart -f values-staging.yaml
  - smoke: POST /v1/campaigns/.../launches with test idempotency key → expect 202

stage: deploy-prod
  - manual approval OR progressive: 10% → 50% → 100%
  - DB migration job runs BEFORE new pods take traffic (additive migrations only)
```

**Additive migration example:**

```sql
ALTER TABLE launch ADD COLUMN suppress_version INT NOT NULL DEFAULT 0;
```

Old code ignores column; new code uses it.

---

## 5. Observability

### 5.1 Structured logging

```json
{
  "timestamp": "...",
  "level": "INFO",
  "tenantId": "acme",
  "campaignId": "cmp_44",
  "launchId": "ln_100",
  "attemptId": "att_9f2",
  "message": "esp_send_ok",
  "durationMs": 142
}
```

**Never** log full phone/email body in prod.

### 5.2 Metrics (Prometheus / New Relic)

| Metric | Type |
|--------|------|
| `launch_accept_total` | counter |
| `launch_dropout` | gauge (accepted - terminal) |
| `esp_request_duration_seconds` | histogram |
| `consumer_lag_seconds` | gauge |
| `budget_reserve_conflict_total` | counter |

### 5.3 Tracing

- Inject `traceparent` at API.
- Outbox relay copies trace context to message headers.
- Worker continues span on consume.

### 5.4 Alerts (business-first)

1. Dropout > 0 for 5 min (P1)
2. Consumer lag > drain SLO (P2)
3. ESP error rate > 5% (P2)
4. VIN suppress lag p99 > 30s (P1 for automotive)

Your **order dropout** story is the template.

---

## 6. AWS pieces

| Service | Use in TAMC study design |
|---------|--------------------------|
| EKS | Run API + workers |
| ECR | Images |
| S3 | Audience export files, vendor cost reconciliation CSV |
| RDS Postgres | Managed OLTP (if not self-hosted) |
| Secrets Manager | ESP API keys |
| ALB + WAF | Public ingress (or Akamai in front) |

---

## 7. Edge (your APIGEE project)

- OAuth2 / JWT validation at gateway.
- Rate limits per API key (dealer).
- Request size limits (bulk payloads → separate upload to S3 + reference in launch).

Service **re-validates** tenant claim vs resource.

---

## 8. Production operations checklist

- Runbook: “consumer lag growing” → scale workers, check poison message, check ESP limit.
- Feature flags: disable SMS channel without deploy.
- Kill switch: pause all launches for tenant (support tool).

---

## 9. Interview questions

- Blue/green vs rolling? (Rolling with maxUnavailable 0 is common on EKS.)
- How migrate DB with zero downtime? (Expand-contract pattern.)
- What did New Relic show during last incident? (Have one real or realistic story.)
