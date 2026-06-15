# Med (med2004x)

**Backend & Systems Engineer • SaaS Founder**  
*Designing, building, and operating high-availability distributed systems, multi-tenant databases, and asynchronous execution pipelines.*

---

### Core Systems & SaaS Architectures

#### Mahall — Global Multi-Tenant E-commerce Infrastructure
A case study in treating multi-vendor marketplace coordination as a distributed systems problem rather than a single storefront application.
*   **The Problem**: Synchronous inventory checkout and payment settlement lead to oversells, split-shipment complexity, and payout reconciliation bottlenecking as seller catalogs scale.
*   **The Solution**: An event-driven architecture that separates consumer checkout from downstream reservation, fulfillment, and payout workflows.
*   **Key Engineering Decisions**:
    *   Decoupled catalog and inventory writes to isolate merchant onboarding spikes from checkout paths.
    *   Modeled payment intents independently of orders to support partial captures and delayed payouts.
    *   Ledger-driven payout architecture that triggers only after fulfillment confirmation.
*   **Tech Stack**: Go, PostgreSQL, Redis, Next.js, Docker, Cloudflare R2.
*   **Case Study / Code**: [`mahall-case-study`](https://github.com/med2004x/mahall-case-study)

#### Lead Source — Distributed Scraping & Playwright Automation Control Plane
A data scraping and enrichment system organized around a Python control plane and isolated execution surfaces.
*   **The Problem**: Blocking HTTP endpoints during long-running browser scraping or external data enrichment causes timeouts, socket exhaustion, and unreliable task progression.
*   **The Solution**: A worker-backed system with task state isolation, separate execution routes for scraping, enrichment, and outreach, and an integrated Playwright automation tier.
*   **Key Engineering Decisions**:
    *   Integrated a remote Playwright browser pool to handle complex dynamic enrichment.
    *   Routed all source execution via a unified pipeline contract to normalize static, API, and browser-assisted inputs.
    *   Asynchronous execution queue managed through Redis, allowing the control API to return immediate job acceptance acknowledgments.
*   **Tech Stack**: Python, Playwright, PostgreSQL, Redis, Prometheus, Grafana.
*   **Case Study / Code**: [`lead-source-architecture-case-study`](https://github.com/med2004x/lead-source-architecture-case-study)

---

### The Engine Room: Distributed Components & Utilities

A portfolio of production-grade services demonstrating deep systems integration, transactional integrity, and operational safety.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Multisurface API Gateway                           │
│              (declarative YAML routing, rate limits, health probes)         │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
            ┌──────────────────────────┴──────────────────────────┐
            ▼                                                     ▼
┌───────────────────────┐                             ┌───────────────────────┐
│  Multitenant Engine   │                             │ Transactional Outbox  │
│  (FastAPI, PG Schemas │                             │ (Atomic Order State & │
│   & SKIP LOCKED)      │                             │  Outbox Relays in Go) │
└───────────┬───────────┘                             └───────────┬───────────┘
            │                                                     │
            ▼                                                     ▼
┌───────────────────────┐                             ┌───────────────────────┐
│ Custom Domain Autom.  │                             │  Go Full-Text Search  │
│ (DNS TXT Verification │                             │ (Worker Inverted      │
│  Stateful Workflows)  │                             │  Index Document DB)   │
└───────────────────────┘                             └───────────────────────┘
```

#### 1. Transactional Outbox Relay ([`transactional-outbox`](https://github.com/med2004x/transactional-outbox))
*   *Go / SQLite / zerolog / Docker Compose*
*   **Problem**: In distributed systems, writing to a database and publishing to an event broker in separate steps risks partial failure (e.g. order saved but message never published).
*   **Solution**: Implements the transactional outbox pattern in Go. Business state and outbox events commit atomically in a single SQLite transaction, backed by a background poller that publishes messages with exponential backoff and a dead-letter path.
*   **Key Detail**: Employs an explicit delivery ledger in SQLite, making retry history and failures queryable directly via SQL rather than hidden in logs.

#### 2. Custom Domain Automation ([`custom-domain-automation`](https://github.com/med2004x/custom-domain-automation))
*   *Go / DNS Provider API / zerolog / Docker Compose*
*   **Problem**: Automating tenant custom domains is error-prone. The handoff between tenant registration, DNS validation (TXT record checks), and SSL provisioning often hangs or requires manual support.
*   **Solution**: A stateful custom domain engine. Registers target hostnames, generates TXT verification tokens, and polls DNS records asynchronously.
*   **Key Detail**: Built with a DNS provider interface to abstract the registrar, isolating the domain progression state machine from third-party API configurations.

#### 3. Multitenant PostgreSQL Engine ([`multitenant-engine`](https://github.com/med2004x/multitenant-engine))
*   *Python / FastAPI / asyncpg / Pydantic / PostgreSQL 16*
*   **Problem**: Shared table multi-tenancy runs the risk of tenant data leaks and makes custom backups or schemas impossible, while separate databases add significant infrastructure overhead.
*   **Solution**: A schema-isolated FastAPI server. Shared control metadata lives in a `public` catalog, while tenant-owned business data is routed dynamically into separate PostgreSQL schemas (`tenant_<slug>`).
*   **Key Detail**: Schema migrations run asynchronously in a worker pool. Bootstrapping workers claim pending migrations using `SELECT ... FOR UPDATE SKIP LOCKED` to scale horizontally.

#### 4. Redis Background Task Processor ([`background-task-processor`](https://github.com/med2004x/background-task-processor))
*   *Python / FastAPI / Redis / redis-py / Pydantic*
*   **Problem**: Celery is often too heavy for lightweight background work, but simple in-memory queues lack retry scheduling, failure visibility, and dead-letter queueing.
*   **Solution**: A lightweight Redis-backed task engine. Uses Redis hashes to store task states for cheap status reads, lists for queued work, and sorted sets for scheduled retries.
*   **Key Detail**: Preserves attempt history and traceback data inside the Redis metadata hash for ease of operator debugging.

#### 5. Go Inverted Index Full-Text Search ([`go-fulltext-search`](https://github.com/med2004x/go-fulltext-search))
*   *Go / Custom Inverted Index / zerolog / Docker Compose*
*   **Problem**: Operating a full Elasticsearch cluster is complex for small catalog/document search needs, but SQL `LIKE` queries lack token-ranking, stemming, and concurrent scaling.
*   **Solution**: A self-contained document search engine written in Go. Accepts document payloads over HTTP, runs tokenization and indexing via a bounded worker channel, and exposes search retrieval endpoints.
*   **Key Detail**: Index mutations are acknowledged only *after* index validation, ensuring search consistency matches database state.

#### 6. Kafka Event Pipeline Ingestion ([`kafka-event-pipeline`](https://github.com/med2004x/kafka-event-pipeline))
*   *Python / FastAPI / confluent-kafka / Pydantic*
*   **Problem**: High-throughput ingestion pipelines must guarantee delivery without crashing under bad payloads or broker outages.
*   **Solution**: A Python event ingestion service with built-in retry envelopes and a local/production transport fallback.
*   **Key Detail**: Encodes retry timing details directly inside the event envelope, allowing topic history to serve as the single source of truth for retry schedules.

#### 7. Multisurface API Gateway Proxy ([`multisurface-api-gateway`](https://github.com/med2004x/multisurface-api-gateway))
*   *Go / yaml.v3 / zerolog / Docker Compose*
*   **Problem**: Fragmented upstream microservices replicate security, API key validation, rate-limiting, and health-checking, creating inconsistent boundaries.
*   **Solution**: A central reverse proxy written in Go. Fronts all services behind uniform auth, rate limits, and health checks.
*   **Key Detail**: Uses active background health-check loops against upstream endpoints to dynamically update routing topology without process reboots.

#### 8. Telemetry & Observability Stack ([`observability-stack`](https://github.com/med2004x/observability-stack))
*   *Go / Prometheus / OpenTelemetry / zerolog*
*   **Problem**: Instrumentation is often added too late. When outages occur, static logs fail to pinpoint system latencies or pinpoint dependency failures.
*   **Solution**: A Go service pre-instrumented with Prometheus RED-style metrics, OpenTelemetry spans, and structured logging.
*   **Key Detail**: Decouples business logic from tracing tools using an injectable telemetry component. Exposes separate `/healthz` (liveness) and `/readyz` (readiness) endpoints.

---

### Technical Index

| Layer | Technology / Pattern | Details |
| :--- | :--- | :--- |
| **Languages** | Go, Python, Rust, TypeScript, C | Concurrent service engines, scripts, systems integration |
| **Databases** | PostgreSQL, SQLite, Redis | Schema isolation, transactional relays, cached configuration |
| **Message Brokers** | Apache Kafka, Redis Pub/Sub | Resilient event ingestion, async backoff envelopes |
| **Edge & Proxy** | Go Reverse Proxy, Cloudflare, CDN | Dynamic YAML routes, rate limiting, SSL automation |
| **Telemetry** | Prometheus, OpenTelemetry, Grafana | RED metrics, distributed tracing, health check topologies |
| **Virtualization** | Docker, Docker Compose, VPS | Portable deployments, container isolated environments |

---

### Engineering Philosophy

*   **Simplicity and Readability**: Software systems are expensive to maintain. Simple, readable code is preferred over clever, abstract solutions.
*   **Product-Minded Engineering**: Software is written to serve a concrete product or business outcome. Deep understanding of billing limits, feature gating, and operational costs drives implementation.
*   **Production Readiness**: A service is not complete when it compiles. It is complete when it is instrumented with RED metrics, liveness/readiness probes, and clean error handling.
*   **Continuous Ownership**: Autonomy demands full-lifecycle accountability, from initial architectural design down to server resource configuration and on-call operations.

---

### Contact & Links

*   **GitHub**: [github.com/med2004x](https://github.com/med2004x)
*   **Email**: [med@med2004x.com](mailto:med@med2004x.com)
*   **LinkedIn**: [linkedin.com/in/med2004x](https://linkedin.com/in/med2004x)
