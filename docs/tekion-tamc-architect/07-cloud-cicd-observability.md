# Cloud-native deployment, CI/CD, and observability (detailed)

**Examples from:** B2B ordering on **EKS** (**P1/P2**), **Jenkins / GitLab CI**, **New Relic** dropout (**P1**), **Grafana**, edge **APIGEE/Akamai/42Crunch** (**P8**), **SRE Triaging** log correlation (**P10**).

Catalog: [11-your-projects-catalog.md](./11-your-projects-catalog.md).

---

## 1. Container image (bulk batch or domain API)

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/bulk-batch-api.jar app.jar
USER nonroot
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]
```

Config: `SPRING_DATA_CASSANDRA_CONTACT_POINTS`, Rabbit URI, New Relic license via env/secret—not in image.

---

## 2. Kubernetes (EKS) — names from your platform

| Deployment | Project | Notes |
|------------|---------|-------|
| `cxp-graphql` / order aggregate | **P1** | Readiness includes Cassandra |
| `bulk-batch-api` | **P2** | maxUnavailable 0 on rollouts |
| `bulk-line-worker` | **P2** | No Ingress; consumes Rabbit |
| `sre-triaging-api` | **P10** | OpenSearch client timeouts |

**Worker graceful shutdown:** on SIGTERM, stop consuming, finish current **line** or chunk, ack message, exit—`terminationGracePeriodSeconds` 60+.

**HPA:** scale **bulk-line-worker** on Rabbit queue depth / consumer lag, not CPU alone.

---

## 3. Probes (lesson from production reviews)

- **Readiness:** can reach Cassandra + Rabbit (**P2**).
- **Liveness:** JVM responsive—**do not** ping Cassandra on liveness (DB blip restarted all workers → backlog spike).

---

## 4. CI/CD (Jenkins / GitLab CI)

```text
mvn verify + Testcontainers (Cassandra or Postgres for Agentic tests)
→ docker build → push ECR
→ deploy staging
→ smoke: POST bulk-batches test idempotency key → 202
→ prod progressive rollout
```

Migrations: additive columns on `bulk_batch` / `agent_run` (**P9**) before traffic shift.

---

## 5. Observability

### 5.1 Structured logs (bulk line)

```json
{
  "tenantId": "b2b-smb",
  "batchId": "bat_9001",
  "lineId": "line_0042",
  "orderId": "ord_771",
  "traceId": "abc123",
  "message": "line_processing_complete",
  "durationMs": 210
}
```

No full cart PII in log line—board rule from **P8/P12**.

### 5.2 Metrics

| Metric | Source |
|--------|--------|
| Order/batch dropout | **New Relic** custom metric (**P1**) |
| Rabbit queue depth | Broker / Grafana |
| PEGA/credit timeout rate | Adapter metric |
| GraphQL p99 | **P1** aggregate |

### 5.3 Tracing

`traceId` from APIGEE (**P8**) through bulk API → worker → order domain—support used this for “line stuck.”

### 5.4 SRE Triaging (P10)

LangGraph node queries **OpenSearch** by `traceId` + time window—**read-only evidence**, not control plane.

### 5.5 Alerts (business-first)

1. Dropout > 0 for 5 minutes
2. Bulk queue depth above SLO
3. Credit adapter error rate > threshold during enrollment peak (**P5**)

---

## 6. AWS

| Service | Your use |
|---------|----------|
| EKS | Run ordering + bulk workers |
| ECR | Images |
| S3 | Large bulk file staging if partners uploaded payload to bucket first (if applicable in your design) |
| EC2 | Legacy or node pools |

---

## 7. Edge (P8)

**APIGEE:** OAuth, quota per partner, WAF integration with Akamai.

Service still validates `tenant_id` on batch against token claims—gateway does not replace authorization.

---

## 8. Interview story

Pick **real or realistic:** “Dropout alert fired; trace showed lines accepted at API but worker could not reach PEGA; we extended circuit breaker and cleared backlog after PEGA recovery—guardrail: DLQ depth alert.”
