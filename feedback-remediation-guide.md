# Feedback Remediation Guide

> **Panel feedback:** Strong on HLD/LLD, Java ecosystem, and architecture approach — but need **deeper** expertise in:
> 1. Java runtime fundamentals
> 2. ORM (JPA/Hibernate)
> 3. Messaging platforms
> 4. Kubernetes architecture
> 5. Complex problem-solving depth

Use this guide **before** your next round. Pair with spoken practice — these topics fail when answers stay at "I know what GC is" without runtime mechanics, failure modes, and tuning trade-offs.

**Study path:** 5 days × 90 min (see [interview-study-plan.md](./interview-study-plan.md#feedback-remediation-5-day-plan)).

---

## Table of Contents

1. [Java Runtime Fundamentals](#1-java-runtime-fundamentals)
2. [ORM — JPA & Hibernate](#2-orm--jpa--hibernate)
3. [Messaging Platforms (Deep)](#3-messaging-platforms-deep)
4. [Kubernetes Architecture](#4-kubernetes-architecture)
5. [Complex Problem-Solving Scenarios](#5-complex-problem-solving-scenarios)
6. [Self-Assessment (Remediation)](#6-self-assessment-remediation)

---

## 1. Java Runtime Fundamentals

### Mental model (draw this)

```
┌─────────────────────────────────────────────────────────┐
│                    JVM Process                            │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐ │
│  │ Class Loader│  │ Bytecode    │  │ JIT Compiler     │ │
│  │ (loading)   │→ │ Interpreter │→ │ (C1/C2, hot code)│ │
│  └─────────────┘  └─────────────┘  └──────────────────┘ │
│                                                          │
│  ┌──────────────── Heap ──────────────────────────────┐ │
│  │ Young: Eden → Survivor S0/S1                      │ │
│  │ Old (Tenured)                                      │ │
│  │ Metaspace (class metadata, off-heap-ish)           │ │
│  └────────────────────────────────────────────────────┘ │
│  ┌──────── Thread stacks (per thread) ───────────────┐ │
│  │ Local vars, operand stack, frame pointers          │ │
│  └────────────────────────────────────────────────────┘ │
│  GC Threads │ JIT Threads │ Application Threads        │
└─────────────────────────────────────────────────────────┘
```

### Q: Walk through JVM memory areas and what breaks at scale.

> **Detailed Answer:** The JVM splits memory into **heap** (objects), **thread stacks** (per-thread frames), and **metaspace** (class metadata). In production microservices, 90% of incidents I debug relate to heap or thread exhaustion — not "Java is slow."
>
> **Heap — Young generation:** New objects allocate in **Eden**. Minor GC copies live objects to **Survivor** spaces; after several GC cycles, long-lived objects promote to **Old gen**. High allocation rate (e.g. creating millions of short-lived JSON DTOs per minute) drives frequent minor GCs — watch **allocation rate** in JFR, not just heap size.
>
> **Heap — Old generation:** Holds long-lived objects — connection pools, caches, domain aggregates kept in memory. When Old gen fills, **major/full GC** runs. G1 does mixed collections; ZGC/Shenandoah target low pause. Symptom: periodic latency spikes matching GC pause charts in Datadog.
>
> **Metaspace:** Class metadata (not the heap in classic sense). Leaks happen with dynamic class generation — Spring proxies, Groovy, some serialization libraries, hot-reload in dev. Symptom: `OutOfMemoryError: Metaspace` after deploys with many new beans.
>
> **Thread stacks:** Each thread has a stack (default ~1MB on Linux). `OutOfMemoryError: unable to create native thread` when you spawn too many threads (thread-per-request without pool) or set `-Xss` too large × thousands of threads.
>
> **Off-heap / direct memory:** NIO buffers, Netty, some DB drivers. Not in heap but counted against container memory limit — pod OOMKilled while heap looks fine.
>
> **Architect takeaway:** Set container memory limit = heap (`-Xmx`) + metaspace + thread stacks + native/direct + headroom (~25%). Never set `-Xmx` to 100% of container limit.

---

### Q: Explain how Garbage Collection works — G1 vs ZGC for a payment service.

> **Detailed Answer:** GC traces **reachable objects** from GC roots (thread stacks, static fields, JNI refs) and reclaims unreachable ones. The cost is **pause time** (stop-the-world phases) vs **throughput** (CPU spent on GC vs app work).
>
> **G1 (default Java 9+):** Divides heap into regions. Targets predictable pause times via `-XX:MaxGCPauseMillis` (default 200ms). Good general-purpose choice for Spring Boot services with mixed object lifetimes. **When to use:** Most microservices, batch + API combined workloads, teams without dedicated JVM tuning.
>
> **ZGC / Shenandoah:** Concurrent collectors aiming for **sub-10ms pauses** even on large heaps (multi-GB). Trade slightly more CPU and complexity. **When to use:** Payment authorization path with strict p99 < 50ms, trading, real-time bidding, any SLA where 200ms GC pause is unacceptable.
>
> **Parallel GC:** Max throughput, long pauses. Batch ETL, offline report generation — not interactive APIs.
>
> **Tuning I actually do in production:**
> - `-Xms` = `-Xmx` (avoid heap resize pauses)
> - Container limits: Java 10+ `UseContainerSupport` — verify heap respects cgroup limit
> - **Don't** tune 20 flags blindly — capture JFR during peak, look at `jdk.GarbageCollection`, `jdk.ObjectAllocationInNewTLAB`
> - If **humongous objects** dominate G1 logs → investigate large byte arrays (JSON responses, PDF generation)
>
> **Spoken trade-off:** *"For our payment API I'd start G1 with equal Xms/Xmx and JFR baselines. If p99 latency still shows 150ms GC spikes at peak, I'd POC ZGC on the same workload before adding more pods."*

---

### Q: What is the Java Memory Model (JMM) and why do architects care?

> **Detailed Answer:** JMM defines **visibility and ordering** of reads/writes across threads — not just "mutex locks everything."
>
> Without proper synchronization, one thread may see stale values of a field another thread updated. The `volatile` keyword ensures writes are visible immediately to other threads (no cache reordering for that field). `synchronized` and `java.util.concurrent` primitives provide mutual exclusion + visibility.
>
> **Why architects care:**
> - **Double-checked locking** on singletons broke before `volatile` was understood — still appears in legacy code.
> - **ConcurrentHashMap** vs `HashMap` — never share plain HashMap across threads in a connection pool callback.
> - **Spring `@Async`** and `@Transactional` — work runs on different threads; thread-local transaction context doesn't magically cross async boundaries without `TransactionTemplate` or reactive propagation.
> - **False sharing** — rarely day-one issue but appears in high-frequency counters (use `LongAdder` vs `AtomicLong` under contention).
>
> **Interview line:** *"I don't implement locks daily, but I review code for shared mutable state in singleton beans, async boundaries, and cache structures under concurrent load."*

---

### Q: JIT compilation — why does Java "warm up" and how does it affect Kubernetes?

> **Detailed Answer:** Bytecode starts in the **interpreter** (slow). Hot methods get compiled by **C1** (fast compile, quick optimization) then **C2** (aggressive optimization) after enough invocations. Peak performance arrives after **warmup** — often minutes of production traffic.
>
> **Implications:**
> - **Cold start after deploy:** First N requests slower — readiness probe should include warmup (load critical classes, hit DB pool, JIT hot paths) or use **startup probes** with longer `failureThreshold`.
> - **Rolling deploy:** Every new pod cold — canary helps warm before full cutover.
> - **GraalVM native image:** AOT compile — fast start, less peak JIT gain; good for Lambda/short-lived jobs, not always for max-throughput long-running APIs.
> - **Microbenchmark lies:** JMH warms up; one-shot benchmarks mislead.
>
> **K8s link:** HPA on CPU during deploy may see high CPU on new pods (JIT + GC) — don't scale down old pods until new ones pass readiness **and** latency canary is green.

---

### Q: Thread pools in Spring Boot — how do you size them and what goes wrong?

> **Detailed Answer:** Spring Boot uses Tomcat's thread pool for HTTP (default max ~200). Separate pools exist for `@Async`, `ForkJoinPool.commonPool()`, JDBC driver internals, and HikariCP connection pool.
>
> **Sizing framework:**
> - **HTTP threads:** Bound by concurrent requests. If each request holds a DB connection, `maxThreads` should not exceed `HikariCP maxPoolSize` significantly — otherwise threads block waiting for connections.
> - **HikariCP:** Rule of thumb `connections = (core_count * 2) + effective_spindle_count` for traditional DB — for SSD/cloud DB, often 10–30 per instance is enough; **too many connections** hurt DB more than help app.
> - **CPU-bound work:** Don't run on HTTP threads — use bounded `ExecutorService` or offload to worker service; pool size ≈ CPU cores.
> - **I/O-bound:** More threads than cores is OK — they block on I/O.
>
> **Failure modes:**
> - Thread pool exhausted → requests queue → timeout cascade
> - Connection pool smaller than threads → threads block on `getConnection()` → looks like "app hang"
> - `ForkJoinPool` starvation when blocking calls inside `parallelStream()`
>
> ```java
> // HikariCP — production-minded defaults
> spring.datasource.hikari.maximum-pool-size=20
> spring.datasource.hikari.connection-timeout=3000
> spring.datasource.hikari.leak-detection-threshold=60000
> ```

---

### Q: Class loading and Spring Boot fat JAR — what happens at startup?

> **Detailed Answer:** Classes load on demand: **loading → linking (verify, prepare, resolve) → initialization** (static blocks run). Spring Boot scans `@Component`, creates bean definitions, wires dependencies — thousands of classes load at startup.
>
> **Fat JAR:** `BOOT-INF/classes` + nested jars in `BOOT-INF/lib`. Custom `LaunchedURLClassLoader` loads nested jars. Startup time = class loading + Spring context + auto-config conditions + DB migration (Flyway).
>
> **Architect optimizations:**
> - Lazy init (`spring.main.lazy-initialization=true`) — faster start, slower first request; good for dev, careful in prod
> - Trim dependencies — unused starters pull transitive jars
> - Spring Boot 3 native / CDS (Class Data Sharing) for faster startup on K8s
> - **Don't** put huge static initialization in `@PostConstruct` on critical path

---

### Java Runtime — Quick Fire

| Question | 30-second answer |
|----------|------------------|
| Eden vs Old gen? | New objects in Eden; survivors promote to Old; full GC on Old |
| Why `-Xms = -Xmx`? | Avoid heap resize pauses; predictable memory for K8s limits |
| G1 vs ZGC? | G1 balanced pauses; ZGC ultra-low pause for latency-sensitive |
| `volatile` vs `synchronized`? | `volatile` visibility only; `synchronized` mutual exclusion + visibility |
| Metaspace OOM? | Too many classes — proxies, dynamic codegen, classloader leak |
| Pod OOM but heap fine? | Direct memory, native threads, or limit too tight vs off-heap |
| JIT warmup? | Interpreter first; hot code compiled; affects deploy latency |

---

## 2. ORM — JPA & Hibernate

### Mental model

```
Controller → Service (@Transactional) → Repository (JPA) → Hibernate Session → JDBC → DB
                      │
                      ├── Persistence Context (1st level cache, per session)
                      ├── Lazy proxies (load on access)
                      └── Flush / SQL generation at transaction boundary
```

### Q: Explain JPA vs Hibernate vs Spring Data — what layer owns what?

> **Detailed Answer:** **JPA** is the specification (interfaces: `EntityManager`, annotations `@Entity`, `@OneToMany`). **Hibernate** is the implementation (Session, dirty checking, SQL generation, caching). **Spring Data JPA** is a repository abstraction (`JpaRepository`) that reduces boilerplate — it still runs Hibernate underneath.
>
> As architect I care that teams don't treat Spring Data as magic:
> - `findAll()` without pagination on 1M rows is a production incident
> - `@Transactional` on service layer defines session boundary
> - Custom `@Query` bypasses some optimizations but still goes through Hibernate
>
> **Decision:** JPA/Hibernate for domain-heavy CRUD with relationships. **JdbcTemplate / jOOQ** when you need full SQL control (complex reports, bulk updates). **No ORM** for simple key-value or event append logs.

---

### Q: What is the N+1 problem and how do you fix it in production?

> **Detailed Answer:** **N+1** happens when you load N parent entities (1 query), then accessing a lazy collection triggers N additional queries — 1 + N round trips.
>
> ```java
> // BAD — 1 query for orders + N queries for each order's items
> List<Order> orders = orderRepo.findByUserId(userId); // SELECT orders
> for (Order o : orders) {
>   o.getItems().size(); // SELECT items WHERE order_id = ?  (× N)
> }
> ```
>
> **Fixes (pick based on context):**
> 1. **JOIN FETCH** — single query with join (watch cartesian product on multiple bags)
>    ```java
>    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.userId = :uid")
>    ```
> 2. **@EntityGraph** — declarative fetch plan on repository method
> 3. **Batch fetching** — `hibernate.default_batch_fetch_size=16` — secondary queries batched IN (...)
> 4. **DTO projection** — query only columns needed; no entity graph
> 5. **Denormalized read model** — CQRS: OpenSearch/Redis for list views
>
> **Detection:** Hibernate `statistics`, `spring.jpa.show-sql` in staging, APM JDBC span count (one HTTP request → 200 SQL queries).
>
> **Trade-off:** JOIN FETCH on large graphs → huge result sets, memory spike. Sometimes 2 queries (orders + batch items by order IDs) beats one giant join.

---

### Q: Lazy vs Eager loading — what do you default to and why?

> **Detailed Answer:** **Default: LAZY** on all associations. Eager loading pulls related data every time — accidental performance death when someone adds `@OneToMany(fetch=EAGER)` and every `findById` loads entire object graph.
>
> **Lazy pitfalls:**
> - **LazyInitializationException** — access association outside `@Transactional` session (common in REST layer after service returns entity)
> - **Fix:** Don't return entities to controller — return DTOs inside transaction; or `@Transactional` on read method with `OpenEntityManagerInView` (discouraged — hides boundary leaks)
>
> **When EAGER is OK:** Tiny, always-needed association (e.g. `Order` → `Currency` enum table with 10 rows) — rare; still prefer explicit fetch in query.

---

### Q: Explain Hibernate caching — 1st level, 2nd level, and when caching hurts.

> **Detailed Answer:**
>
> | Cache | Scope | Enabled? | Use |
> |-------|-------|----------|-----|
> | **1st level (Persistence Context)** | Per `EntityManager`/session | Always | Identity map within transaction |
> | **2nd level** | Shared across sessions (entity cache) | Opt-in + provider (EhCache, Redis) | Rarely changed reference data |
> | **Query cache** | Cached query result IDs | Opt-in | Very read-heavy identical queries |
>
> **2nd level cache criteria:** Entity changes infrequently (country codes, product categories), high read ratio, acceptable staleness, cluster-safe provider if multiple pods.
>
> **When caching hurts:**
> - Stale reads on inventory/stock — **never** 2nd-level cache mutable inventory
> - Memory pressure across many pods (each JVM holds copy unless centralized Redis)
> - Debugging "ghost data" — hard to trace
>
> **Architect preference:** Application-level Redis cache with explicit TTL and key design beats opaque Hibernate 2nd level for most microservices.

---

### Q: `@Transactional` — propagation, read-only, and common production bugs.

> **Detailed Answer:** Spring's `@Transactional` binds a Hibernate `Session` to the thread and demarcates commit/rollback.
>
> **Propagation:**
> - `REQUIRED` (default) — join existing tx or create new
> - `REQUIRES_NEW` — suspend current, new tx — use for audit log that must commit even if outer fails
> - `NOT_SUPPORTED` — suspend tx — rare external calls
>
> **readOnly=true:** Hint to Hibernate — no dirty checking, can optimize flush; use on all read services.
>
> **Common bugs:**
> 1. **Self-invocation** — `@Transactional` on private method or same-class call bypasses proxy → no transaction
> 2. **Long transactions** — HTTP call inside `@Transactional` holds DB connection and row locks
> 3. **Rollback only on unchecked** — checked `Exception` doesn't rollback by default (`rollbackFor`)
> 4. **Isolation surprises** — default READ_COMMITTED; phantom reads on concurrent inventory unless explicit locking (`@Lock(PESSIMISTIC_WRITE)` or optimistic `@Version`)
>
> ```java
> @Transactional(readOnly = true)
> public OrderDto getOrder(String id) {
>   return orderRepo.findById(id)
>       .map(this::toDto)  // map inside tx — lazy fields safe
>       .orElseThrow();
> }
> ```

---

### Q: Optimistic vs pessimistic locking for inventory — architect decision.

> **Detailed Answer:**
>
> **Optimistic (`@Version` column):**
> - Read entity, business logic, update with version check `UPDATE ... WHERE id=? AND version=?`
> - If 0 rows updated → `OptimisticLockException` → retry or fail
> - **Best when:** Low contention, few conflicts (most products)
>
> **Pessimistic (`SELECT FOR UPDATE`):**
> - Lock row at read time — blocks other writers
> - **Best when:** High contention (last seat, flash sale SKU), short transaction
> - **Risk:** Deadlocks, lock wait timeout under spike — combine with queue at edge
>
> **Hybrid at scale:** Redis `SET NX` seat lock (short TTL) + DB confirm with optimistic version on commit — Redis absorbs spike, DB enforces truth.

---

### ORM — Quick Fire

| Question | Answer |
|----------|--------|
| N+1? | 1 + N queries from lazy collections; fix: fetch join, batch, DTO |
| LazyInitializationException? | Lazy load outside session; use DTO inside `@Transactional` |
| 2nd level cache? | Shared entity cache; only stable reference data |
| Open Session In View? | Extends session to web layer — convenient but hides leaks |
| `ddl-auto=update` in prod? | Never — use Flyway/Liquibase |
| JPA vs jOOQ? | JPA for domain model; jOOQ for complex SQL/reporting |

---

## 3. Messaging Platforms (Deep)

### Q: Kafka internals — partitions, offsets, consumer groups (architect depth).

> **Detailed Answer:** A Kafka **topic** is split into **partitions** — ordered, append-only logs. Each partition is replicated across brokers (ISR = in-sync replicas). **Producers** write to a partition (key hash → partition, or sticky partitioner). **Consumers** in a **consumer group** each own a subset of partitions — max parallelism = partition count per group.
>
> **Offsets:** Each consumer tracks offset per partition. Commit **after** processing (at-least-once) or use transactional consume-produce (exactly-once within Kafka). **Replay:** reset offset to earlier position — same consumer group or new group reads history.
>
> **Key architect decisions:**
> - **Partition key:** `orderId` keeps all events for one order ordered; bad key (constant) → one hot partition
> - **Partition count:** Hard to reduce later; plan for peak consumer parallelism
> - **Retention:** `retention.ms` — audit vs disk cost; compacted topics for changelog (Kafka Streams, Debezium)
> - **Min ISR + `acks=all`:** Durability vs latency trade-off
>
> ```
> Topic: orders (6 partitions)
>   P0 P1 P2 P3 P4 P5
> Consumer group "payment":
>   Instance A → P0, P1
>   Instance B → P2, P3
>   Instance C → P4, P5
> New group "analytics" → independent offset, reads all from beginning or latest
> ```

---

### Q: RabbitMQ — acknowledgments, prefetch, and poison messages.

> **Detailed Answer:** RabbitMQ pushes messages to consumers; **ACK** tells broker to delete (or requeue).
>
> | ACK mode | Behavior | Risk |
> |----------|----------|------|
> | **Manual ACK** | ACK after successful processing | Must ACK or NACK; crash before ACK → redelivered |
> | **Auto ACK** | ACK on deliver | Message lost if consumer crashes mid-process |
> | **NACK requeue=false** | Dead-letter or drop | Use with DLX |
>
> **Prefetch (`basicQos prefetch=10`):** Limits unacked messages per consumer — prevents one slow consumer hoarding thousands. **Architect tuning:** Low prefetch (1–10) for fair work distribution; higher for batch consumers.
>
> **Poison message:** Bad payload causes repeated failures → infinite redelivery. **Fix:** DLX (dead-letter exchange) after `x-max-retries` or reject without requeue; alert on DLQ depth; idempotent consumer.
>
> **Publisher confirms:** `confirmSelect()` — broker acks message persisted to disk/queue before producer continues — use for critical events (payment charged).

---

### Q: RabbitMQ vs Kafka — deep comparison for architect panel.

> **Detailed Answer:** Beyond the one-liner, panels want **operational** depth:
>
> | Scenario | RabbitMQ | Kafka |
> |----------|----------|-------|
> | Task queue (one worker processes job) | Native queue model | Possible but awkward (competing consumers per partition) |
> | Event log / audit / replay | Poor (message deleted on ACK) | Native |
> | Multiple independent readers | Need fanout exchange + multiple queues | Consumer groups + log retention |
> | Ordering | Per queue | Per partition |
> | Backpressure | Consumer prefetch + queue depth alarm | Consumer lag metric |
> | Ops | Easier cluster smaller scale | KRaft/ZK, partition rebalance, ISR monitoring |
> | Delayed / priority messages | Plugins / TTL DLX | Not native (use separate scheduling) |
>
> **Hybrid:** Outbox → Kafka for event bus; RabbitMQ for per-service work queues (send email job). **Don't** use Kafka as job queue without understanding consumer stuck on poison partition.

---

### Q: Exactly-once processing — realistic architect answer.

> **Detailed Answer:** True end-to-end exactly-once across DB + broker is **hard**. Production pattern:
>
> 1. **At-least-once delivery** (broker + consumer retries)
> 2. **Idempotent consumer** — dedup table `(messageId)` or business key `(orderId, eventType)`
> 3. **Transactional outbox** — DB write + outbox row in same transaction; relay publishes to broker
> 4. **Kafka exactly-once** — `idempotent producer` + `transactions` for consume-transform-produce within Kafka only
>
> **Spoken line:** *"I design for at-least-once plus idempotency. Exactly-once inside Kafka for stream processing; cross-system I use outbox and dedup keys."*

---

### Q: Schema evolution with Avro + Schema Registry.

> **Detailed Answer:** JSON events break consumers silently when fields change. **Avro/Protobuf** with **Schema Registry** enforces compatibility:
> - **BACKWARD:** new consumer reads old data (add optional fields with defaults)
> - **FORWARD:** old consumer reads new data
> - **FULL:** both
>
> Breaking change (rename field without alias) → new subject version rejected in CI. **Architect process:** event schema is API — review in PR, contract tests for consumers.

---

### Messaging — Quick Fire

| Question | Answer |
|----------|--------|
| Consumer lag? | Kafka: offset behind high-water mark; scale consumers ≤ partitions |
| Hot partition? | Bad partition key; skewed traffic |
| `acks=1` vs `all`? | 1 = leader ack; all = min ISR ack — durability |
| RabbitMQ quorum queues? | Raft-based HA queues; prefer over classic mirrored |
| Rebalance storm? | Too frequent consumer join/leave; session timeout tuning |
| Outbox pattern? | Same DB tx as business write; relay to broker |

---

## 4. Kubernetes Architecture

### Control plane mental model (draw this)

```
┌──────────────── Control Plane (master) ─────────────────┐
│  API Server ← all kubectl, controllers, kubelet talk here │
│  etcd ← cluster state (desired + actual)                  │
│  Scheduler → assigns pod to node                          │
│  Controller Manager → ReplicaSet, Deployment, Node ctrl   │
└───────────────────────────────────────────────────────────┘
         │
    ┌────┴────┬────────────┐
    ▼         ▼            ▼
 Worker    Worker       Worker
 (kubelet) (kubelet)    (kubelet)
   Pods      Pods         Pods
```

### Q: Explain Kubernetes networking — Pod, Service, Ingress, CNI.

> **Detailed Answer:** Every pod gets an IP from the **CNI** (Calico, Cilium, AWS VPC CNI). Pods talk pod-to-pod across nodes via overlay or native routing — **flat network**, no NAT between pods by default.
>
> **Service (ClusterIP):** Stable virtual IP + DNS (`order-service.default.svc.cluster.local`) → kube-proxy or dataplane (iptables/IPVS/eBPF) routes to healthy pod endpoints. **Selector** matches pod labels.
>
> **Service types:**
> - **ClusterIP** — internal only
> - **NodePort** — expose on node IP (dev/debug)
> - **LoadBalancer** — cloud LB provisions (EKS → ELB/NLB)
>
> **Ingress / Ingress Controller:** HTTP routing layer (path/host → service). Needs controller (nginx, ALB Ingress Controller). TLS termination at ingress.
>
> **NetworkPolicy:** Firewall for pods — default allow all; restrict `payment-service` only accepts from `order-service` namespace. **Architect:** zero-trust internal mesh often uses NetworkPolicy + service mesh mTLS.
>
> **DNS:** CoreDNS resolves service names — apps use service name, not pod IP.

---

### Q: How does a Deployment rolling update work and how do you achieve zero downtime?

> **Detailed Answer:** Deployment owns ReplicaSet; change image → new ReplicaSet created, scales up new pods while scaling down old per `strategy`:
> ```yaml
> strategy:
>   type: RollingUpdate
>   rollingUpdate:
>     maxSurge: 1        # extra pods during rollout
>     maxUnavailable: 0  # never reduce below desired during update
> ```
> **Zero downtime requirements:**
> 1. **Readiness probe** — new pod ready before old terminated
> 2. **PreStop hook** — `sleep 5` + graceful shutdown (SIGTERM, drain HTTP server)
> 3. **PodDisruptionBudget** — min available during voluntary disruptions
> 4. **HPA** — don't fight rollout with aggressive scale-down
>
> **Canary / Argo Rollouts:** Route 5% traffic to new version via service mesh or weighted ingress; promote on error rate / latency SLO.

---

### Q: Resource requests, limits, and QoS — why pods get OOMKilled or throttled.

> **Detailed Answer:**
>
> | Setting | Meaning |
> |---------|---------|
> | **requests** | Scheduler placement; guaranteed CPU share |
> | **limits** | Max CPU throttle / memory kill |
>
> **QoS classes:**
> - **Guaranteed** — requests = limits → highest priority, last evicted
> - **Burstable** — requests < limits → common for Java apps
> - **BestEffort** — no requests → first evicted
>
> **Java on K8s:** Set container memory limit; `-Xmx` ~70–75% of limit; rest for metaspace, threads, direct memory. **CPU:** Java benefits from consistent CPU — avoid too-low CPU request causing throttling and long GC.
>
> **OOMKilled:** Container exceeded memory limit — check heap dump, direct buffers, native thread count.

---

### Q: HPA, VPA, Cluster Autoscaler — when to use each.

> **Detailed Answer:**
> - **HPA (Horizontal Pod Autoscaler):** Scale pod count on CPU, memory, or custom metrics (Prometheus adapter: queue depth, Kafka lag). **Architect:** custom metrics beat CPU for I/O-bound Java services.
> - **VPA (Vertical):** Adjust requests/limits per pod — rarely auto in prod without careful rollout; good recommendations.
> - **Cluster Autoscaler:** Add/remove nodes when pods can't schedule. **Watch:** scale-up delay, pod pending timeout.
>
> **KEDA:** Event-driven autoscaling — scale consumers on RabbitMQ queue length or Kafka lag directly.

---

### Q: Service mesh (Istio/Linkerd) — why would an architect add it?

> **Detailed Answer:** Mesh adds sidecar proxy (Envoy) per pod for:
> - **mTLS** between services without app changes
> - **Traffic splitting** — canary, A/B
> - **Retries, timeouts, circuit breaking** at mesh layer (don't duplicate blindly in app + mesh)
> - **Observability** — automatic trace spans per hop
>
> **Costs:** Sidecar memory (~50–100MB per pod), CPU, operational complexity, debugging harder. **Use when:** 20+ services, security mandates mTLS, frequent canary deploys. **Skip when:** Small cluster, team lacks mesh ops — ALB + app-level resilience is enough.

---

### Q: ConfigMaps, Secrets, and externalized config at scale.

> **Detailed Answer:** **ConfigMap** for non-sensitive config; **Secret** for credentials (base64 in etcd — enable encryption at rest). **External Secrets Operator** syncs from AWS Secrets Manager / Vault. **12-factor:** config in environment, not in image. **Architect:** separate config per env via Helm values or Kustomize overlays; never bake prod secrets into Jenkins.

---

### Kubernetes — Quick Fire

| Question | Answer |
|----------|--------|
| etcd role? | Cluster state store; backup critical |
| kubelet role? | Runs pods on node; reports health |
| Liveness vs readiness? | Liveness restart; readiness remove from service endpoints |
| Headless service? | No ClusterIP; direct pod DNS for StatefulSet |
| StatefulSet vs Deployment? | Stable identity, ordered rollout, persistent volume per pod |
| PodDisruptionBudget? | Min pods available during node drain / upgrade |
| Why not liveness check DB? | DB blip kills all pods → outage amplification |

---

## 5. Complex Problem-Solving Scenarios

Practice these as **15-minute spoken walkthroughs**: clarify → options → trade-offs → decision → failure modes.

### Scenario A: Payment double-charge under retries

**Prompt:** Payment gateway times out; clients retry; some customers double-charged.

**Strong answer structure:**
1. **Root cause:** Timeout ≠ failure; client retry without idempotency key
2. **Immediate:** Stop retries at gateway if possible; reconcile gateway ledger vs internal DB
3. **Fix:** Idempotency key `orderId + paymentIntent` stored in Redis/DB; gateway supports same key → same result
4. **Saga:** Payment activity idempotent; Temporal workflow state prevents duplicate charge activity
5. **Prevention:** Contract tests for timeout+retry; canary with synthetic slow gateway

---

### Scenario B: Kafka consumer lag spikes to millions during deploy

**Prompt:** After deploy, `inventory-consumer` lag grows; inventory stale; oversell risk.

**Strong answer:**
1. **Triage:** Single consumer group? Partition count? Rebalance during deploy?
2. **Causes:** Slow processing (N+1 in new code), poison message stuck, rebalance storm, reduced pod count
3. **Mitigate:** Scale consumers (≤ partitions), skip bad offset to DLQ after max retries, feature flag old version
4. **Long-term:** Static membership, cooperative rebalance, consumer lag alert, load test consumer with production message size

---

### Scenario C: Java service p99 latency 3s after traffic 2x — no errors

**Prompt:** CPU 60%, errors low, but checkout p99 terrible.

**Strong answer (layer by layer):**
1. **GC logs / JFR** — full GC or humongous allocations?
2. **Thread dump** — all threads blocked on `getConnection()`? Hikari pool exhausted?
3. **APM traces** — N+1 queries (200 SQL per request)?
4. **Downstream** — payment p99 elevated? DNS intermittent?
5. **K8s** — CPU throttling from low limits? Noisy neighbor on node?
6. **Fix examples:** Increase pool only after DB capacity check; batch fetch; ZGC POC; raise CPU request

---

### Scenario D: Zero-downtime migration monolith DB → service-owned DB

**Prompt:** Extract `inventory` from shared PostgreSQL without downtime.

**Strong answer:**
1. **Strangler:** Inventory service reads/write new path behind feature flag
2. **Dual-write** period — monolith + new service both write (temporary inconsistency risk)
3. **CDC (Debezium)** or outbox sync monolith → new DB until cutover
4. **Expand-contract** schema changes only
5. **Cutover:** Flip read to new service; stop monolith writes; verify reconciliation job
6. **Rollback plan:** Feature flag back to monolith path

---

### Scenario E: Flash sale — 100K users, 500 seats

**Strong answer:**
1. **Edge:** Queue-it / token at CDN; reject overload before JVM
2. **Seat map:** Redis `SET NX` per seat with TTL lock; atomic Lua for adjacent seats
3. **DB:** Confirm with pessimistic or optimistic lock; unique constraint on `(show_id, seat_id)` where status=BOOKED
4. **Async checkout:** Don't hold HTTP 30s — return reservation token, complete via WebSocket/poll
5. **Failure:** Lock TTL frees ghost locks; idempotent payment confirm

---

### Scenario F: Multi-region active-active for orders

**Strong answer:**
1. **Clarify:** Active-active reads vs writes? RPO/RTO?
2. **Hard part:** Inventory consistency across regions — avoid dual-write without coordination
3. **Patterns:** Region-scoped inventory shards; global orders with region leader; CRDB/Spanner for global consistency (cost)
4. **Kafka:** MirrorMaker / cluster linking; consumer region affinity
5. **Trade-off:** Active-passive simpler; active-active needs conflict resolution (last-write-wins bad for money)

---

## 6. Self-Assessment (Remediation)

Score 0–3 before and after the 5-day plan. **Target: ≥2 on all rows, ≥3 on bold rows.**

| # | Topic | Score | Can explain without notes? |
|---|-------|-------|----------------------------|
| 1 | JVM heap generations + metaspace | | |
| 2 | **G1 vs ZGC trade-off with numbers** | | |
| 3 | JIT warmup + K8s deploy impact | | |
| 4 | Thread pool vs connection pool sizing | | |
| 5 | **N+1 detection and 3 fixes** | | |
| 6 | Lazy vs eager + LazyInitializationException | | |
| 7 | `@Transactional` propagation bugs | | |
| 8 | **Optimistic vs pessimistic locking** | | |
| 9 | Kafka partitions + consumer groups | | |
| 10 | **RabbitMQ ACK, prefetch, DLX** | | |
| 11 | Outbox + idempotent consumer | | |
| 12 | K8s Service vs Ingress vs NetworkPolicy | | |
| 13 | **Rolling update + zero downtime** | | |
| 14 | Requests/limits + Java memory in pod | | |
| 15 | HPA on custom metrics (queue lag) | | |
| 16 | Scenario: double-charge retry | | |
| 17 | Scenario: consumer lag spike | | |
| 18 | Scenario: p99 latency no errors | | |

---

*After scoring, run [Mock Round 4](./mock-interview-practice.md#round-4--feedback-remediation-60-minutes) timed.*
