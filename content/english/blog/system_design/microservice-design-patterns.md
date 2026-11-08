+++
date = '2025-06-19T12:00:00+10:00'
draft = false
title = 'Microservice Design Patterns'
tags = ['Microservices', 'System Design', 'Architecture', 'Saga', 'CQRS', 'API Gateway', 'Service Discovery']
summary = 'Comprehensive reference on all 44+ microservice design patterns catalogued by Chris Richardson plus patterns from Microsoft.'
+++

Microservice design patterns — catalogued primarily by Chris Richardson in _Microservices Patterns_ (44+ patterns across 8 categories) — provide battle-tested solutions to recurring problems in distributed systems. This reference covers all pattern groups, their sub-patterns, when to apply each and the trade-offs involved, followed by supplementary Microsoft cloud patterns.

![](../img/png/chris-richardson-microservice.png)

---

## Decomposition Patterns

The foundational challenge: how to split a monolith into services.

### Decompose by Business Capability

Define services around business capabilities (e.g., Order Management, Inventory, Shipping) rather than technical layers. Each service owns a distinct business function. Drives alignment with Conway's Law — organisation structure mirrors system architecture.

**Use when** you have clear organizational boundaries aligned with business units. Works well when capabilities have low interdependency.

**Don't use when** capabilities are tightly coupled or share lots of data — you'll end up with chatty services and distributed transactions.

### Decompose by Subdomain (DDD)

Use Domain-Driven Design's strategic patterns — bounded contexts, aggregates and domain events — to identify service boundaries. Each bounded context maps to one service.

**Use when** the domain is complex and you have DDD-experienced teams. Produces services with high cohesion and loose coupling.

**Don't use when** the domain is simple CRUD; the overhead of DDD isn't justified.

### Strangler Fig

Gradually replace monolith functionality with microservices by routing specific requests to new services. The monolith is "strangled" over time until it can be decommissioned.

**Use when** migrating an existing monolith incrementally with zero downtime. Requires a routing layer (API gateway or reverse proxy) that can split traffic per-feature.

**Don't use when** you need a full rewrite — strangler fig is explicitly for incremental, coexistence-based migration.

### Anti-Corruption Layer (ACL)

Implement a facade or adapter between a modern application and a legacy system to prevent legacy domain concepts from leaking into the new system. Translates between the two domain models.

**Use when** integrating with a legacy system with a different domain model, especially during strangler fig migration.

**Don't use when** there is no legacy system or the integration surface is trivial.

---

## Communication Patterns

The most diverse group, spanning synchronous, asynchronous and hybrid approaches.

### API Gateway

A single entry point that routes requests to appropriate services, handles cross-cutting concerns (auth, rate limiting, logging, TLS termination) and may aggregate responses.

Microsoft further decomposes this into:

- **Gateway Routing** — route requests to services based on path/headers
- **Gateway Aggregation** — combine multiple backend responses into a single client response
- **Gateway Offloading** — offload shared functionality (TLS, auth, SSL termination) to the gateway

**Use when** clients need a unified API, you want to offload cross-cutting concerns or you need protocol translation (e.g., external REST to internal gRPC).

**Don't use when** the added hop (latency) is unacceptable or for simple systems where clients can call services directly.

### Backend for Frontend (BFF)

Dedicate a separate API gateway per client type (web BFF, mobile BFF, IoT BFF). Each BFF knows the specific data and interaction needs of its client.

**Use when** client types have substantially different API requirements or when mobile teams need to evolve their API independently of web.

**Don't use when** clients are homogeneous — a single API gateway suffices.

### Service Discovery

In a microservice architecture, service instances have dynamically assigned network locations. Service discovery patterns handle locating them at runtime.

- **Client-Side Discovery** — the client queries a service registry to get the list of available instances and load-balances across them directly
- **Server-Side Discovery** — the client makes a request to a load balancer or API gateway, which queries the registry and forwards the request
- **Service Registry** — a highly available database (Consul, Eureka, ZooKeeper, Kubernetes DNS) that stores the network locations of service instances
- **Self Registration** — a service instance registers itself with the registry on startup and deregisters on shutdown
- **Third Party Registration** — a third-party registrar (e.g., Kubernetes, Nomad) handles registration/deregistration without the service needing to know about the registry

**Use when** service locations are dynamic (container orchestration, auto-scaling) and you need to decouple clients from specific instance addresses.

**Don't use when** using a service mesh (Envoy/Istio) — the mesh handles discovery transparently at the sidecar level.

### Remote Procedure Invocation (RPI)

Synchronous communication via REST, gRPC or Thrift. gRPC offers strong contracts via protobuf, streaming and bidirectional flow.

**Use when** low-latency, real-time request-response semantics are needed. gRPC especially for high-performance internal service-to-service calls.

**Don't use when** services must remain loosely coupled and independently evolvable — async messaging is a better fit.

### Domain-Specific Protocol

Use a custom protocol tailored to a specific domain (e.g., SMTP for email, RTMP for streaming). Avoid generic protocols when domain semantics justify a specialised wire format.

### Async Messaging / Event-Driven

Services communicate via messages published to a broker (Kafka, RabbitMQ, AWS SQS/SNS). The producer and consumer are decoupled in time and space.

**Use when** you need loose coupling, event-driven workflows, or integration across heterogeneous systems.

**Don't use when** request-response semantics are required and the added latency of async is unacceptable.

### Publisher-Subscriber

An intermediary (topic/exchange) fans out messages to all subscribers. Enables broadcast of domain events to multiple consumers.

**Use when** a single event must trigger multiple independent actions (e.g., "OrderPlaced" triggers billing, shipping, notifications).

**Don't use when** you need point-to-point delivery with competing consumers — use a queue instead.

### Claim Check

Split a large message into a claim check (reference) and payload stored elsewhere. The message bus carries only the reference.

**Use when** message payloads exceed broker size limits or you want to avoid transferring large payloads through the bus.

### Competing Consumers

Multiple consumer instances read from the same queue and compete for messages. Each message is processed by exactly one consumer.

**Use when** you need to scale throughput horizontally by adding more consumer instances.

**Don't use when** message ordering is critical (competing consumers break ordering guarantees).

### Priority Queue

Priority queue ensures high-priority messages are processed before low-priority ones.

**Use when** different classes of workload have different urgency levels.

---

## Data Management Patterns

Each service owns its data; no shared database across services. Services communicate only via their API.

**Use when** you need independent deployability, schema evolution per service and isolated failure domains. The default choice for microservices.

**Don't use when** reporting or cross-service queries become too complex — you may need a CQRS view or an aggregated reporting database (with careful ownership).

### Database per Service

The canonical microservice data pattern. Each service has its own private database — no direct access by other services.

### Saga

Manage data consistency across multiple services without distributed transactions (which are not feasible in microservices).

**Choreography**: Each service publishes events after local transactions; downstream services listen and react. Decentralised, simple but hard to track.

**Orchestration**: A central coordinator (orchestrator) tells each service what to do and handles compensation on failure. Centralised, easier to monitor but adds a dependency.

**Use when** a business operation spans multiple services and requires eventual consistency with rollback capability.

**Don't use when** strong consistency across services is mandatory (rare in microservices — consider a monolith instead).

### API Composition

Perform a query that retrieves data from multiple services by having an API composer (often the API gateway or a dedicated service) invoke each service and combine the results.

**Use when** data is spread across multiple services and the query can be satisfied with simple joins from a few services.

**Don't use when** the query requires complex joins or aggregations across large datasets — use CQRS with a materialised view instead.

### CQRS (Command Query Responsibility Segregation)

Split service into command and query models, often with separate data stores. Commands use write-optimised storage; queries use read-optimised views (possibly denormalised).

**Use when** reads and writes have asymmetric throughput requirements, when queries are complex (need multiple DB types) or when event sourcing is in play.

**Don't use when** the domain is simple or the consistency lag between models causes user-facing confusion (must tolerate eventual consistency).

### Event Sourcing

Persist state as a sequence of domain events rather than a single current-state row. Reconstruct current state by replaying events.

**Use when** you need a complete audit trail, temporal queries (state at any point in time) or complex event-driven workflows.

**Don't use when** query performance matters (event replay is expensive for high-read systems), the event schema changes frequently or the domain is simple CRUD.

### Transactional Outbox

Persist a domain event in the same local database transaction as the state change. A relay process publishes the event to a message broker. Two primary implementation strategies:

- **Transaction Log Tailing (CDC)** — a process (Debezium, Kafka Connect) tails the database transaction log and publishes changes to the message broker. Zero impact on application code.
- **Polling Publisher** — a background process periodically polls an outbox table for unpublished events and publishes them. Simpler to implement but adds latency and database load.

**Use when** you need reliable event publishing without 2PC (two-phase commit). Guarantees at-least-once delivery.

**Don't use when** your database doesn't support the required primitives (e.g., lack of CDC capability in some hosted DBs).

### Shared Database

Services share a single database (anti-pattern for microservices but used during strangler fig migration).

**Use when** migrating incrementally from a monolith — keep a shared DB temporarily while splitting services.

**Don't use when** building new microservices — it couples deployability, creates contention and prevents independent scaling.

### Materialized View

Precompute and store denormalised views of data across multiple services to serve read queries efficiently.

**Use when** you need to query across service boundaries without complex runtime joins.

### Sharding

Partition a data store horizontally across multiple database instances using a shard key. Not a microservice-specific pattern but essential for data scalability in distributed systems.

---

## Observability Patterns

### Log Aggregation

Centralise logs from all services into a searchable store (ELK, Loki, CloudWatch). Each service writes structured, correlated logs.

**Use when** you have 3+ services and need root-cause analysis across service boundaries.

### Application Metrics

Collect and expose metrics (request rate, error rate, latency percentiles, resource utilisation) from each service. Tools: Prometheus, Micrometer, OpenTelemetry.

**Use when** you need to monitor service health, set up alerts and drive auto-scaling decisions.

### Distributed Tracing

Propagate a trace ID across service calls to reconstruct end-to-end request flows. Tools: Jaeger, Zipkin, OpenTelemetry.

**Use when** latency-sensitive or multi-service operations need per-request debugging.

### Health Check

Expose `/health` (liveness) and `/ready` (readiness) endpoints for load balancers, orchestrators and monitoring systems.

Use with Kubernetes liveness/readiness probes for automatic pod replacement.

### Exception Tracking

Aggregate and alert on exceptions across services using tools like Sentry, Rollbar or Application Insights.

---

## Reliability Patterns

### Circuit Breaker

Wrap calls to external services in a circuit breaker. On repeated failures the circuit "opens" and subsequent calls fail fast (or return a fallback). After a timeout, a "half-open" trial determines if the service recovered.

**Use when** calling remote services that may fail or slow down; prevents cascading failures.

### Bulkhead

Isolate resources (thread pools, connections, memory) per service or client group so failure in one partition doesn't exhaust shared resources.

**Use when** you must guarantee that a failing client or service doesn't starve others.

### Retry with Exponential Backoff

Retry transient failures with increasing delay + jitter to avoid thundering herd on recovery.

**Use when** failures are transient (network blips, DB connection timeouts).

**Don't use when** failures are non-transient (400 Bad Request) — retrying is wasteful.

### Timeout

Bound the maximum time a service waits for a response from a downstream dependency. Fail fast rather than hanging indefinitely.

**Use when** every remote call — always. Timeouts prevent resource exhaustion from slow or hung dependencies.

### Rate Limiting / Throttling

Control the rate of requests a service accepts. Rate limiting is applied at the client side to stay within limits; throttling is server-side enforcement that returns 429 Too Many Requests.

**Use when** you need to protect a service from overload, enforce SLAs or control costs.

### Queue-Based Load Leveling

Use a queue as a buffer between a producer (task sender) and consumer (task processor) to smooth intermittent heavy loads.

**Use when** workload spikes are common and you want to protect the backend from being overwhelmed.

### Scheduler Agent Supervisor

Coordinate a set of distributed actions. A scheduler initiates the workflow, an agent performs each action and a supervisor monitors outcomes, retrying or compensating on failures.

**Use when** you need a reliable, distributed batch process or long-running workflow.

### Leader Election

Elect one instance among a cluster to coordinate tasks (e.g., which instance processes the CDC stream). Tools: ZooKeeper, etcd, Kubernetes leader election.

**Use when** only one instance should perform a task at a time and you need automatic failover.

---

## Security Patterns

### Access Token

Clients present an access token (opaque or JWT) with each request. The API gateway or service validates the token before processing.

### JWT (JSON Web Token)

A self-contained access token format carrying claims (user ID, roles, expiry) signed by the issuer. Stateless verification — no database lookup needed.

**Use when** you need stateless, distributed authentication across services.

### Federated Identity

Delegate authentication to an external identity provider (Azure AD, Okta, Auth0) rather than building your own auth system.

### Gatekeeper

A dedicated, hardened host instance that validates and sanitises all requests before forwarding them to internal services. Reduces attack surface.

### Valet Key

Provide clients with a restricted, time-limited token that grants direct access to a specific resource (e.g., Azure SAS, presigned S3 URL) without exposing the master key.

---

## Deployment Patterns

### Service Instance per Container

Each service instance runs in its own container (Docker), orchestrated by Kubernetes, Nomad or similar. The standard deployment model.

### Service Instance per VM

Each service instance runs in its own VM. Stronger isolation but heavier and slower to provision.

### Multiple Service Instances per Host

Run multiple service instances (or different services) on the same host/VM. Simpler operations but weaker isolation and risk of resource contention.

### Blue/Green Deployment

Run two identical environments (blue and green). Route traffic to green; when green is healthy, switch traffic. Enables zero-downtime deploys and instant rollback.

### Canary Deployment

Roll out new version to a small subset of users, monitor, then gradually increase traffic. Cuts blast radius of bad releases.

### Serverless

Deploy individual functions (AWS Lambda, Cloud Functions) that scale automatically per request. Good for event-driven workloads and simple APIs.

**Use when** traffic is spiky or unpredictable and cold-start latency is acceptable.

**Don't use when** workloads are steady-state (provisioned servers are cheaper) or functions require long-running compute.

### Deployment Stamps

Deploy multiple independent copies of the entire application stack (including data stores), typically per tenant or per region. Each stamp is fully self-contained.

**Use when** multi-tenant isolation, geo-distribution or per-customer deployment is required.

---

## Testing Patterns

### Consumer-Driven Contract Tests

The consumer defines the expected contract (request/response shape). The provider verifies it satisfies all consumer contracts in CI. Tools: Pact, Spring Cloud Contract.

**Use when** multiple consumers depend on a single service and you need to detect breaking changes early.

### Consumer-Side Contract Tests

The consumer validates that the provider API matches the expected contract. Run on the consumer side in CI; often paired with a mock provider.

### Service Component Test

Test a single service in isolation by stubbing its dependencies. Fast, cheap and run in CI on every commit.

### Service Integration Contract Test

Test the integration between a service and its external dependencies using real infrastructure (or lightweight test containers).

### End-to-End Test

Test a cross-service flow end-to-end, including infrastructure. Slow and brittle — use sparingly for critical paths only.

---

## Cross-Cutting Patterns

### Microservice Chassis

Start each service from a standardised foundation (framework, logging, metrics, configuration, health checks). Reuse across the organisation to reduce boilerplate.

### Externalised Configuration

All configuration (DB URLs, feature flags, secrets) lives outside the service code — environment variables, config server (Consul, Spring Cloud Config) or Kubernetes ConfigMaps/Secrets.

**Use when** configuration varies across environments (dev/staging/prod) or changes without a redeploy.

### Ambassador

A helper sidecar that handles network concerns (retries, circuit breaking, authentication, observability) on behalf of the application container. Placed on the same pod/host as the service.

**Use when** you want to offload cross-cutting network concerns without modifying the application code or adopting a full service mesh.

### Service Mesh (Sidecar)

Offload cross-cutting concerns (service discovery, retries, circuit breaking, mTLS, observability) to a sidecar proxy (Envoy, Linkerd, Istio). Application code stays focused on business logic.

**Use when** the organisation has many services and wants platform-level traffic management without changing application code.

**Don't use when** the operational complexity of running a mesh outweighs the benefits (small clusters, few services).

---

## Additional Microsoft Cloud Patterns

The following patterns from the [Microsoft Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/patterns/) address concerns not covered by the core Chris Richardson catalog but are relevant to production microservices:

| Pattern                            | Purpose                                                                                         |
| ---------------------------------- | ----------------------------------------------------------------------------------------------- |
| **Ambassador**                     | Offload network concerns to a helper sidecar (already covered above)                            |
| **Cache-Aside**                    | Load data into a cache on-demand; the cache is populated lazily on read miss                    |
| **Claim Check**                    | Split large messages into reference + blob (already covered above)                              |
| **Competing Consumers**            | Scale message processing horizontally (already covered above)                                   |
| **Compute Resource Consolidation** | Consolidate multiple tasks into a single compute unit to reduce cost                            |
| **Deployment Stamps**              | Per-tenant/region deployment copies (already covered above)                                     |
| **Gatekeeper**                     | Hardened request validation proxy (already covered above)                                       |
| **Geode**                          | Deploy backend services across geographically distributed nodes; any node can serve any request |
| **Leader Election**                | Coordinate single-active-instance tasks (already covered above)                                 |
| **Materialized View**              | Precomputed cross-service query views (already covered above)                                   |
| **Priority Queue**                 | Prioritised message processing (already covered above)                                          |
| **Queue-Based Load Leveling**      | Buffer spikes with a queue (already covered above)                                              |
| **Scheduler Agent Supervisor**     | Reliable distributed batch workflow (already covered above)                                     |
| **Sequential Convoy**              | Process related messages in order without blocking other message groups                         |
| **Sharding**                       | Horizontal data partitioning (already covered above)                                            |
| **Static Content Hosting**         | Serve static assets directly from cloud storage (CDN + blob storage)                            |
| **Throttling**                     | Server-side request rate enforcement (already covered above)                                    |
| **Valet Key**                      | Scoped, time-limited direct resource access (already covered above)                             |

### Cache-Aside

On a read miss, the application loads data from the database into the cache (Redis, Memcached). Subsequent reads hit the cache. Writes invalidate the cache entry.

**Use when** read-heavy workloads benefit from low-latency cache hits.

**Don't use when** data changes so frequently that cache invalidation overhead outweighs benefits.

### Geode

Deploy backend services across multiple geographically distributed nodes, each capable of handling any client request. Provides low-latency access regardless of client location.

**Use when** you have a global user base and need sub-100ms response times from every region.

### Sequential Convoy

Process related messages in a defined order without blocking other groups. Each group maintains its own ordering sequence (e.g., all events for a single order ID are processed in order).

**Use when** ordering within a partition matters but cross-partition ordering does not.

### Static Content Hosting

Serve static assets (images, CSS, JS) directly from cloud blob storage (Azure Blob, S3) fronted by a CDN rather than from application servers.

---

## Pattern Selection Guide

| Problem                         | Primary Pattern                             |
| ------------------------------- | ------------------------------------------- |
| How to split services?          | Decompose by subdomain (DDD)                |
| How to migrate?                 | Strangler Fig                               |
| How to find services?           | Service Registry + Client/Server Discovery  |
| How to route traffic?           | API Gateway                                 |
| Different clients?              | BFF                                         |
| Distributed transaction?        | Saga                                        |
| Reliable event publishing?      | Transactional Outbox (CDC or Polling)       |
| Query across services?          | API Composition or CQRS + Materialized View |
| Audit trail / temporal queries? | Event Sourcing                              |
| Read/write asymmetry?           | CQRS                                        |
| Prevent cascading failures?     | Circuit Breaker + Bulkhead                  |
| Protect from overload?          | Rate Limiting / Queue-Based Load Leveling   |
| Isolate concerns from code?     | Ambassador or Service Mesh                  |
| Client-independent deploys?     | Consumer-Driven Contracts                   |
| Zero-downtime deploys?          | Blue/Green or Canary                        |
| Global low-latency?             | Geode                                       |

---

## References

- Richardson, C. — _Microservices Patterns_ (Manning, 2018; 2nd ed. 2027)
- [microservice.io — Patterns by Chris Richardson](https://microservice.io)
- [Microsoft Azure Architecture Center — Cloud Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/patterns/)
- [Microsoft — .NET Microservices Architecture Guidance](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/)
- Newman, S. — _Building Microservices_ (O'Reilly, 2nd ed.)
