# System Design Roadmap — 6 Modules, In Depth

Organized around six modules. Within each module, concepts are still taught as
a causal chain (**A creates a problem, B solves it but introduces a new
problem**) because that's how the ideas actually connect — but the modules
themselves map to a standard system design curriculum so you can track
progress against it. Every module ends with **"→ leads to"**, naming why the
next module's problems appear once you've internalized this one.

---

## Module 1 — Client-Server Architecture, HTTP, APIs and Databases

### 1.1 What happens when you type a URL and hit enter
DNS resolution → TCP handshake → TLS handshake → HTTP request → server
processes → response → browser renders. Memorize this flow cold — it's the
skeleton every later module attaches a box to (load balancer goes between TCP
and your server; cache goes between "processes" and the database; CDN
intercepts before DNS even reaches your origin).

### 1.2 The client-server model, in detail
- **Request/response cycle** and why HTTP is fundamentally a pull protocol
  (server can't push to a client uninvited — this is *why* WebSockets/SSE/
  long-polling exist for anything real-time, see 1.5 and Module 5's chat app).
- **HTTP methods** (GET/POST/PUT/PATCH/DELETE) and the idempotency distinction
  that matters for retries: GET/PUT/DELETE are idempotent, POST is not — this
  single fact drives how you design retry logic in Module 4.
- **Status codes** as a taxonomy, not trivia: 2xx success, 3xx redirect
  (caching implications), 4xx client error (your API contract was violated),
  5xx server error (your system failed — these are the ones your on-call
  cares about, see Module 4's observability).
- **Headers**: `Content-Type`, `Cache-Control` (ties directly to Module 2's
  caching), `Authorization`, `ETag`/`If-None-Match` for conditional requests.
- **HTTP is stateless** — the server remembers nothing about you between
  requests. This one fact is *why* cookies/sessions/tokens exist (1.8), why
  load balancers can freely round-robin (Module 2), and why "stateless
  services" is treated as a hard requirement rather than a style preference.

### 1.3 Numbers every engineer should know
You don't need exact figures memorized, you need the *ratios*, because every
"should I cache this / should I colocate this / should I paginate this"
decision in later modules is really this table in disguise:

| Operation | Rough latency |
|---|---|
| L1 cache reference | ~1 ns |
| RAM access | ~100 ns |
| SSD random read | ~100 μs (~1000x slower than RAM) |
| Same-datacenter network round trip | ~0.5 ms |
| Cross-region network round trip | ~100-150 ms |
| Disk seek (spinning) | ~10 ms |

RAM is roughly 1000x faster than SSD; same-datacenter is roughly 100x faster
than cross-continent. Those two ratios alone justify most of Module 2.

### 1.4 Vertical scaling and its ceiling
A single machine has finite CPU/RAM/disk/network. You can buy a bigger
machine (vertical scaling) — it's simple and needs no architecture change —
but cost grows faster than capacity, there's a hardware ceiling, and it's a
single point of failure. This ceiling is the entire reason Module 2 exists.

### 1.5 API paradigms
- **REST** — resource-oriented, maps cleanly onto HTTP verbs/status codes,
  cache-friendly (GET responses are cacheable by default per 1.3's numbers),
  the default choice for public/external APIs.
- **GraphQL** — client specifies exactly the shape of data it wants in one
  round trip. Solves REST's over-fetching (getting fields you don't need) and
  under-fetching (needing 3 REST calls to assemble one screen). Introduces
  its own problems: the N+1 query problem on the server resolving nested
  fields, and losing HTTP-level caching since every query is a POST to one
  endpoint.
- **gRPC / RPC** — binary (protobuf), HTTP/2 multiplexed streaming, low
  overhead. Great for internal service-to-service calls where you control
  both ends and care about latency more than human-readability. Bad for
  public APIs (not browser-native, harder to debug).
- **WebSockets** — a persistent, full-duplex connection instead of
  request/response. Needed whenever the server must push unprompted (chat,
  live notifications, collaborative editing) — this is the direct fix for
  1.2's "HTTP can't push" limitation, and it's the backbone of Module 5's
  chat app design.
- **Webhooks** — inversion of control for server-to-server: instead of you
  polling a third party, they POST to a URL you registered when an event
  happens. Cheap way to get "push" semantics without holding a connection open.

### 1.6 Databases — what they actually give you over a file
Before Module 3's SQL/NoSQL deep dive, know *why* you don't just write JSON
to disk: a database gives you **durability** (write-ahead logging survives a
crash), **concurrent access** without corruption (locking/MVCC), **queryability**
without scanning everything by hand, and (for SQL specifically) **transactional
guarantees** across multiple writes. Every database product is a different
trade-off of these four properties against write throughput and scale — that
trade-off is the entirety of Module 3.

### 1.7 Auth fundamentals (belongs here because it's an API-layer concern)
- **Authentication vs authorization** — who are you, vs what are you allowed
  to do. Keep them conceptually separate even though they often live in the
  same middleware.
- **Session-based auth** — server stores session state, client holds an
  opaque session ID in a cookie. Conflicts with 1.2's statelessness
  requirement unless sessions live in a shared store (Redis) that every
  server instance can read — a second reason (beyond speed) that Module 2's
  cache layer exists.
- **Token-based auth (JWT)** — the client holds a signed, self-contained
  token; the server verifies the signature instead of looking up state. Trades
  server memory for token-revocation difficulty (you can't easily "delete" a
  JWT before it expires without a blocklist, which reintroduces state).
- **OAuth2 / OIDC** — delegated authorization ("let app X access my Google
  contacts without giving X my password"), the standard for third-party
  login and third-party API access.
- **TLS** (from 1.1) covers encryption in transit; encryption at rest matters
  again at Module 3 (encrypted disks/backups) and Module 2 (encrypted cache
  contents for sensitive data).

**→ leads to:** a single client-server pair with one database is the whole
picture so far. The moment traffic exceeds what one server or one database
copy can handle, you need more machines doing the same job — and something
routing traffic between them, plus something absorbing repeat reads before
they ever hit that database. That's Module 2.

---

## Module 2 — Load Balancers, Cache, CDN and Message Queues

### 2.1 Horizontal scaling
Run N copies of your stateless service (1.2 made this legal) instead of one
bigger machine (1.4's ceiling). Cheaper per unit of capacity, no hardware
ceiling, and copies can be added/removed independently — but now a client
needs to know *which* copy to talk to. That's the problem load balancers solve.

### 2.2 Load balancers, in depth
- **L4 (transport layer)** — routes based on IP/port, doesn't inspect HTTP.
  Fast, cheap, protocol-agnostic, but can't make routing decisions based on
  URL path or headers.
- **L7 (application layer)** — terminates and inspects HTTP, can route by
  path/host/header (e.g., `/api/*` to one service, `/static/*` to another),
  do SSL termination, and buffer slow clients. Costs more CPU per request.
- **Algorithms**: round robin (naive, ignores server load), weighted round
  robin (accounts for heterogeneous server capacity), least connections
  (adapts to actual load), IP hash / consistent hashing (same client → same
  server, needed for **sticky sessions** when session state isn't fully
  externalized — the same consistent-hashing technique reappears in Module
  3's sharding and Module 2's own cache-node assignment, one idea, three uses).
- **Health checks** — LB stops routing to a server that fails liveness/
  readiness probes; this is the mechanism underneath Module 4's failover.
- **LB high availability** — the load balancer itself can't be a new single
  point of failure. DNS-based LB (e.g., Route 53 with health checks) or an
  active-passive LB pair is how you avoid that.
- **Where LBs sit**: client → LB → services (classic), but also service → LB
  → service internally (or a service mesh sidecar proxy doing the same job
  at the service-to-service level once you have Module 5-scale microservices).

### 2.3 Statelessness in practice
For an LB to freely send request #2 to a *different* server than request #1,
no server can hold session state in local memory (1.2, 1.7). Session data
either lives in a shared store (Redis, reachable by every instance) or
travels with the client (JWT). This isn't a style preference — it's a
requirement for 2.2 to work at all.

### 2.4 Caching, in depth
- **Why cache** — directly cashes in on 1.3's latency ratios: serving from
  RAM instead of a database round trip is 100-1000x faster and removes load
  from the database entirely (pairing with Module 3's scaling ladder).
- **Cache layers**, each removing load from everything behind it: browser
  cache (client-side, zero network cost on hit) → CDN/edge (2.5) → app-level
  cache (Redis/Memcached in front of the DB) → database-internal buffer pool
  (the DB's own RAM cache of hot pages).
- **Cache patterns**:
  - *Cache-aside (lazy loading)* — app checks cache, on miss reads DB and
    populates cache. Simple, most common; cache can go stale between DB
    writes and the next read.
  - *Write-through* — app writes to cache and DB synchronously. Cache is
    never stale but every write pays cache-write latency.
  - *Write-back (write-behind)* — app writes to cache, cache asynchronously
    flushes to DB. Fast writes, but risk of data loss if the cache node dies
    before flushing.
  - *Read-through* — cache itself is responsible for loading from DB on miss
    (app only ever talks to the cache).
- **Invalidation strategies** — TTL expiry (simple, bounded staleness),
  explicit invalidation on write (immediate but easy to miss a code path),
  versioned/cache-busting keys. "There are only two hard things in computer
  science: cache invalidation and naming things" — because a stale cache is
  the same *kind* of problem as Module 3's replication lag: once a copy of
  data exists anywhere, you're trading freshness for speed.
- **Eviction policies** — LRU (most common, evict least-recently-used),
  LFU (evict least-frequently-used, better for skewed access patterns), FIFO,
  random (cheap, surprisingly competitive).
- **Cache stampede / thundering herd** — when a hot key expires, many
  concurrent requests miss simultaneously and all hammer the DB at once.
  Fixed with request coalescing (only one request repopulates, others wait),
  locking, or jittered TTLs so keys don't all expire in sync.
- **Hot key problem** — one key (a viral post, a celebrity's profile) gets
  disproportionate traffic and can overload a single cache shard even though
  the cluster overall has capacity. Fixed by replicating just that key across
  multiple nodes or adding a local (in-process) cache layer in front of the
  distributed one.

### 2.5 CDN, in depth
- Geographically distributed edge servers (Points of Presence) that cache
  content physically close to the user — directly exploiting 1.3's
  cross-region latency gap.
- **Static content** (images, JS/CSS bundles, video) is the classic use case;
  increasingly CDNs also cache **dynamic content** at the edge with short
  TTLs or edge compute (edge functions).
- **Push vs pull CDN** — pull: CDN fetches from origin on first miss and
  caches it (simpler, slight first-request latency); push: you proactively
  upload content to the CDN (used for content you know will be hot, e.g. a
  new release).
- Controlled via `Cache-Control`/`Expires` headers (ties back to 1.2).
- Side benefit: a CDN absorbs a large fraction of traffic spikes and basic
  DDoS load before it ever reaches your origin — relevant again in Module 4.

### 2.6 Message queues, in depth
- **Why sync calls don't scale for everything** — if service A calls service
  B directly and waits, A is only as available and fast as B. Chain enough
  synchronous calls and one slow dependency takes down the whole request path.
- **Core mechanics** — producer enqueues, consumer dequeues and processes,
  visibility timeout (message is hidden from other consumers while being
  processed, reappears if not acknowledged in time — handles crashed
  consumers).
- **Delivery guarantees**:
  - *At-most-once* — message might be lost, never duplicated (fire-and-forget).
  - *At-least-once* — message is never lost, but might be processed twice
    (the common default; requires your handler to be **idempotent**).
  - *Exactly-once* — the ideal, and nearly a myth in real distributed
    systems: it requires either transactional dedup on the consumer side or
    tight coupling between producer and broker. This connects straight back
    to the same network unreliability that breaks CAP's perfect consistency
    (Module 3) — you can't get a free lunch from an unreliable network either
    way.
- **Dead-letter queues** — messages that repeatedly fail processing get
  routed aside for manual inspection instead of blocking the queue forever.
- **Traditional queue (SQS/RabbitMQ) vs distributed log (Kafka)** — a queue
  deletes a message once consumed; a log retains messages for a retention
  window and lets multiple independent consumer groups replay from any
  offset. Kafka trades simplicity for replayability and much higher
  throughput, at the cost of needing partition-key design (which rhymes with
  Module 3's sharding).
- **Backpressure** — when consumers can't keep up, the queue absorbs the
  burst instead of the burst hitting downstream services directly (a queue
  is itself a backpressure mechanism, tying into Module 4).

### 2.7 Pub/Sub and event-driven architecture
Generalizes the queue from one consumer to many independent subscribers
reacting to the same event, with no coupling between publisher and
subscribers. This is the seed of **event-driven architecture** — services
react to *what happened* rather than being told what to do — used for fan-out
notifications, analytics pipelines, and (in Module 5) social feed fan-out on
post creation.

**→ leads to:** load balancing and caching both make a *single request* go
fast, and queues decouple slow work from the request path. But all of this
still runs on top of a data layer, and you haven't yet decided how that data
layer itself scales, replicates, or survives a machine failure. That's
Module 3.

---

## Module 3 — SQL, NoSQL, Indexing, Replication and Sharding

### 3.1 SQL / relational databases, in depth
- **Relational model** — structured schema, tables with typed columns, joins
  across tables instead of duplicating data.
- **Normalization** — organizing schema to eliminate redundancy (up through
  3NF is enough to know cold); trades write-side cleanliness for read-side
  join cost — the opposite trade NoSQL usually makes.
- **ACID**, each letter is a separate guarantee worth naming individually:
  - *Atomicity* — a transaction's writes all happen or none do.
  - *Consistency* — a transaction moves the DB from one valid state to
    another (schema/constraints always hold).
  - *Isolation* — concurrent transactions don't see each other's
    in-progress writes.
  - *Durability* — once committed, a write survives a crash (this is the
    write-ahead log from 1.6).
- **Isolation levels and the anomalies they prevent** — read uncommitted
  (dirty reads possible) → read committed → repeatable read (prevents
  non-repeatable reads) → serializable (prevents phantom reads too, at the
  cost of throughput). Picking an isolation level is picking how much
  concurrency you give up for correctness.

### 3.2 NoSQL, in depth
Built for horizontal scale and flexible schema from day one, trading
relational guarantees for it. Four families, each good at a different shape:
- **Key-value** (Redis, DynamoDB) — O(1) lookup by key, no query language,
  extremely fast; use when access is always "give me the value for this ID."
- **Document** (MongoDB) — JSON-like documents, flexible schema per
  document, supports querying into nested fields; good when your "entity"
  naturally nests (a user profile with embedded preferences).
- **Wide-column** (Cassandra, HBase) — rows can have different columns,
  optimized for huge write volume and range scans over a partition key; the
  default choice for time-series/event data at scale.
- **Graph** (Neo4j) — nodes and edges as first-class citizens, optimized for
  traversal queries (friend-of-friend, recommendation paths) that would be
  many expensive joins in SQL.
- **BASE** (Basically Available, Soft state, Eventually consistent) as the
  NoSQL-world counterpart to ACID — an explicit acknowledgment that these
  systems favor availability and partition tolerance over immediate
  consistency (this is CAP, 3.6, made into a design philosophy).

### 3.3 Choosing SQL vs NoSQL
The real decision axis: *do you need multi-record transactional consistency
and complex relational queries (SQL), or do you need to scale writes
horizontally from the start and can tolerate denormalized, eventually
consistent data (NoSQL)?* Most real systems are **polyglot persistence** —
Postgres for the transactional core (orders, payments), Redis for
session/cache, Cassandra or DynamoDB for high-volume event data, Elasticsearch
for search — not a single database for everything.

### 3.4 Indexing, in depth
- **B-tree index** — the default; turns an O(n) table scan into an O(log n)
  lookup, and supports range queries (`WHERE age BETWEEN 20 AND 30`) because
  it keeps sorted order. Directly mirrors DSA's binary search intuition.
- **Hash index** — O(1) exact-match lookup, but can't do range queries at all.
- **Composite (multi-column) indexes** — column order matters: an index on
  `(a, b)` serves queries filtering on `a` alone or `a AND b`, but not `b`
  alone.
- **Covering index** — an index that contains every column a query needs, so
  the DB never touches the underlying table row at all.
- **The cost** — every index speeds up reads but slows down writes (the
  index itself must be updated) and takes storage. "Index everything" is
  wrong; index the columns your actual query patterns filter/sort/join on.

### 3.5 Replication, in depth
- **Leader-follower (single writer, many readers)** — the common starting
  point. Solves read scaling (route reads to replicas) and durability (data
  survives if one machine dies).
- **Synchronous vs asynchronous replication** — sync replication waits for a
  follower ack before confirming the write (safer, higher write latency,
  follower failure can block writes); async is faster but risks losing the
  most recent writes if the leader dies before followers catch up.
- **Multi-leader** — multiple nodes accept writes (useful for multi-region
  write locality) but now needs conflict resolution (last-write-wins,
  vector clocks, CRDTs) when two leaders accept conflicting writes for the
  same record.
- **Leaderless / quorum-based** (Dynamo-style, Cassandra) — writes go to W
  replicas, reads from R replicas, and `W + R > N` guarantees at least one
  overlapping node sees the latest write. Tunable per-operation consistency
  is the whole appeal.
- **Replication lag** — followers can fall behind the leader, so a read
  right after a write might return stale data (the **read-your-writes**
  problem — commonly fixed by routing a user's own reads to the leader for a
  short window after they write).

### 3.6 CAP theorem (and PACELC)
Once data is replicated across machines connected by an unreliable network,
you cannot have perfect **C**onsistency, **A**vailability, and **P**artition
tolerance simultaneously the instant a network partition happens. Partition
tolerance isn't optional in a real distributed system, so every distributed
data system is actually choosing, *under partition*, between **CP** (reject
requests to stay consistent — e.g. a majority-quorum system like etcd/Spanner)
and **AP** (serve possibly-stale data to stay available — e.g.
Cassandra/DynamoDB). **PACELC** extends this: even when there's *no*
partition, you still trade **L**atency against **C**onsistency (do you wait
for all replicas to agree, or answer fast from whichever replica you hit?).
This is the lens for evaluating any data-layer decision in this module.

### 3.7 Sharding / partitioning, in depth
Replication scales *reads*; it doesn't help once a single dataset is too
large or too write-heavy for one machine. Sharding splits data across
machines so each holds a subset.
- **Range-based** — shard by a sorted key range (e.g. user IDs 1-1M on shard
  1). Simple, supports range scans, but prone to hotspotting if traffic
  skews toward one range (e.g. newest users, all landing on the last shard).
- **Hash-based** — shard by `hash(key) % N`. Distributes load evenly but
  destroys range-query locality and makes adding a shard expensive (nearly
  everything remaps) — motivating:
- **Consistent hashing** — nodes and keys are placed on a hash ring; adding
  or removing a node only remaps the keys adjacent to it on the ring, not the
  whole keyspace. The same technique load balancers use for sticky routing
  (2.2) and caches use for shard assignment (2.4) — one idea, three uses.
- **Directory-based** — a lookup service maps key → shard explicitly. Most
  flexible (can rebalance arbitrarily), but the directory itself becomes a
  critical, must-be-highly-available component.
- **Choosing a shard key** — the single most consequential sharding decision.
  A bad key (e.g. sharding a multi-tenant system by signup date) creates
  hotspots; a good key (e.g. tenant ID) spreads load and keeps a tenant's
  data on one shard so most queries don't cross shards.
- **Cross-shard queries/joins** — expensive by construction; the usual
  answer is denormalizing data onto the shard that needs it, or doing
  application-level scatter-gather and accepting the latency cost.
- **Resharding pain** — the reason consistent hashing exists at all; without
  it, adding capacity means moving nearly all your data.

### 3.8 The scaling ladder, compressed
Vertical scaling (1.4) → read replicas (3.5, cheap, only helps reads) →
caching (Module 2, cheapest lever, sits in front of everything) → sharding
(3.7, the expensive last resort because it touches your data model and your
queries). Reach for each step only after the previous one is exhausted —
this ordering is itself a common interview signal (Module 6).

**→ leads to:** replication, CAP trade-offs, and sharding all assume you can
tolerate *some* imperfection under load — but you still need to actually
answer "will this hold up," "what happens when traffic spikes 10x," and "what
happens when a shard or a replica dies in production." That's Module 4.

---

## Module 4 — Scalability, Rate Limiting and High Availability

### 4.1 Scalability, defined properly
Scalability isn't "handles more traffic," it's *how gracefully* cost and
latency grow as load grows. Two separate axes matter:
- **Throughput** (requests/sec the system can sustain) vs **latency**
  (time per request) — you can often buy one at the cost of the other
  (batching improves throughput but adds latency per item).
- **Read scaling vs write scaling** — reads scale via replicas/cache (Modules
  2-3); writes scale via sharding (3.7) or async processing (2.6) — they are
  *not* the same problem and need different tools.
- **Back-of-envelope capacity estimation** — the core interview skill (see
  Module 6): given "500M daily active users, each posts twice a day," derive
  QPS (~11,600 writes/sec average, note the peak-to-average ratio can be
  5-10x), storage growth per year, and bandwidth. Getting the *order of
  magnitude* right (is this a "cache it" problem or a "shard it" problem?)
  matters far more than precision.

### 4.2 Rate limiting, in depth
- **Why** — protect a service from being overwhelmed (accidental retry
  storms, abusive clients, a misbehaving internal service) and enforce
  fairness across tenants/users.
- **Algorithms**:
  - *Fixed window counter* — simplest, but allows a burst of 2x the limit
    right at the window boundary (e.g. 100 requests at 11:59:59, another 100
    at 12:00:00).
  - *Sliding window log* — tracks every request timestamp, exact but memory-
    heavy at scale.
  - *Sliding window counter* — approximates the sliding log cheaply by
    weighting the previous window's count — the common production compromise.
  - *Token bucket* — tokens refill at a fixed rate, a request consumes a
    token; naturally allows short bursts up to the bucket size while
    enforcing a long-run average rate. The most commonly used in practice.
  - *Leaky bucket* — requests queue and are processed at a fixed output
    rate, smoothing bursts entirely (no burst allowance, unlike token bucket).
- **Where enforced** — client-side (cooperative, not a real defense),
  per-IP or per-API-key at the API Gateway (1.5/2.2, the natural
  chokepoint), or per-user deeper in the service.
- **Distributed rate limiting** — the counter must be shared across every LB
  ins­tance handling that client, which means it lives in Redis (2.4) —
  and now you have a race condition on concurrent increments, solved with
  atomic operations (`INCR` + `EXPIRE`, or a Lua script for compound checks).

### 4.3 Backpressure and load shedding
When incoming load exceeds capacity, the system needs an explicit policy
rather than falling over: bound queue depth and reject (rather than queue
indefinitely and degrade every request's latency), shed low-priority traffic
first, or serve cached/stale/degraded responses instead of failing outright
(e.g. show a cached recommendation list instead of a personalized one under
load).

### 4.4 Circuit breakers, timeouts, retries
- **Circuit breaker states** — closed (normal, requests pass through) → open
  (recent failures exceeded a threshold, requests fail fast without even
  trying the downstream) → half-open (after a cooldown, let a few requests
  through to test recovery). This stops a slow/failing downstream (2.6's
  sync-call problem) from taking down its callers by queuing up doomed
  requests.
- **Timeouts** — every network call needs one; without it, a hung downstream
  ties up your caller's threads/connections indefinitely.
- **Retries with exponential backoff + jitter** — naive immediate retries
  from many clients synchronize into a **retry storm** that makes an outage
  worse; backoff spreads retries out, jitter (randomizing the backoff)
  prevents them from re-synchronizing.
- **Bulkheading** — isolate resource pools (thread pools, connections) per
  downstream dependency, so one slow dependency can't exhaust resources
  needed to call a different, healthy one.

### 4.5 High availability, in depth
- **Redundancy, no single point of failure** — every layer (LB, service,
  cache, DB) needs more than one instance, or its failure is the system's
  failure.
- **Active-active vs active-passive** — active-active serves traffic from
  all replicas simultaneously (better resource use, harder consistency);
  active-passive keeps a standby idle until failover (simpler, wastes
  capacity, brief failover delay).
- **Multi-AZ vs multi-region** — multi-AZ protects against a data-center-
  level failure with low added latency (same-region round trips, 1.3);
  multi-region protects against a regional failure but reintroduces
  cross-region latency and much harder data consistency (3.5's replication
  lag becomes a cross-continent problem).
- **Availability math** — the "number of 9s" table (99.9% = ~8.7 hrs
  downtime/year, 99.99% = ~52 min/year, 99.999% = ~5 min/year) and why each
  additional 9 gets disproportionately expensive.
- **SLI vs SLO vs SLA** — SLI is the measured metric (e.g. p99 latency), SLO
  is your internal target for it, SLA is the contractual promise to
  customers with consequences for missing it. Know the distinction; it comes
  up whenever "how available does this need to be" is asked (Module 6).

### 4.6 Distributed coordination underpinning HA
- **Consensus (Raft/Paxos)** — how a cluster agrees on a single value/leader
  even when some machines fail or messages are lost. This is the actual
  mechanism underneath 3.5's "leader" in leader-follower replication and
  underneath any service registry staying consistent.
- **Leader election** — a direct application of consensus, used whenever a
  system needs exactly one coordinator (a DB primary, a job scheduler, a
  distributed lock service like ZooKeeper/etcd).
- **Distributed transactions vs sagas** — an atomic operation across
  multiple services/databases classically needs two-phase commit (2PC):
  expensive, fragile, and blocks on a coordinator failure. Microservice
  architectures instead use **sagas** — a sequence of local transactions
  with explicit compensating actions on failure (e.g. "refund payment" as
  the compensation for "reserve inventory" if a later step fails).

### 4.7 Observability — the feedback loop for everything above
You cannot verify that a rate limiter, a circuit breaker, or a failover
actually works by reading code — you need runtime signal:
- **Metrics** — is the system healthy right now (error rate, p50/p99
  latency, saturation).
- **Logs** — what exactly happened for a specific request/error.
- **Distributed tracing** — follow one request across every service/queue/
  DB it touched, essential once Module 2's async paths and Module 5's
  microservices mean no single log file has the whole story.

**→ leads to:** Modules 1-4 are the full vocabulary. Module 5 is where you
prove you can actually combine them into a coherent design, and Module 6 is
where you prove you can explain that design's trade-offs out loud under time
pressure.

---

## Module 5 — Real-World System Design: URL Shortener, Chat App, E-commerce, Social Media

For each design, work through the same five steps every time — the structure
matters more than the specific answer, and it's the exact structure Module 6
expects you to narrate in an interview:

1. **Requirements** — functional (what must it do) and non-functional (scale,
   latency, consistency needs).
2. **Capacity estimate** — QPS, storage, bandwidth (4.1's back-of-envelope math).
3. **API design** — the handful of endpoints/methods that satisfy the
   functional requirements (1.5).
4. **High-level architecture** — draw client → LB → service → cache → DB,
   with async side-paths for queues (this diagram *is* 1.1's mental model,
   with every module's boxes now filled in).
5. **Deep dive** — the one or two hardest sub-problems, and the trade-off
   you're explicitly making.

### 5.1 URL Shortener
- **Functional**: shorten a URL, redirect a short code to the original,
  optional custom aliases/expiry.
- **Non-functional**: redirect latency must be very low; reads vastly
  outnumber writes (read-heavy).
- **Deep dive — ID generation**: base62 encoding of an auto-incrementing ID,
  vs a hash of the URL (collision handling needed), vs pre-generated key
  pools handed out to app servers (avoids a single counter becoming a
  bottleneck — a direct application of 3.7's sharding a write hotspot away).
- **Modules exercised**: 3.4 (key-value lookup, effectively an index),
  3.7 (sharding by short code if the key space is huge), 2.4 (cache hot
  short codes aggressively — classic Pareto traffic distribution).

### 5.2 Chat / Messaging App (WhatsApp/Slack-shaped)
- **Functional**: 1:1 and group messaging, delivery/read receipts, online
  presence, message history.
- **Non-functional**: low latency delivery, messages must not be lost,
  ordering within a conversation matters.
- **Deep dive — real-time delivery**: WebSocket connections held per online
  user (1.5), a connection-to-server mapping service so any backend node can
  find which gateway node holds a given user's socket, message queue (2.6)
  per user/conversation for offline delivery and durability, fan-out to
  multiple devices per user.
- **Deep dive — ordering & consistency**: per-conversation sequence numbers
  or vector clocks so messages render in a consistent order even if delivery
  arrives out of order across servers.
- **Modules exercised**: 2.6/2.7 (queues for durability and offline fan-out),
  3.5 (replication for durability), 4.6 (ordering/consistency guarantees),
  1.5 (WebSockets).

### 5.3 E-commerce Platform (Amazon-shaped)
- **Functional**: product catalog/search, cart, checkout/payment, order
  tracking, inventory.
- **Non-functional**: strong consistency required for inventory and payment
  (can't oversell, can't double-charge); catalog browsing can tolerate
  eventual consistency and heavy caching.
- **Deep dive — inventory & checkout**: this is the module's clearest ACID
  case (3.1) — reserving inventory and charging a card must be atomic across
  services, which is exactly the saga pattern from 4.6 (reserve inventory →
  charge payment → confirm order, with compensating "release inventory" and
  "refund" steps if a later step fails).
- **Deep dive — catalog/search**: denormalized read models (3.2's document
  store) and a search index (Elasticsearch) kept eventually consistent with
  the transactional order/inventory system — a concrete instance of 3.3's
  polyglot persistence.
- **Modules exercised**: 3.1/3.6 (strong consistency where money/inventory
  is involved), 4.6 (sagas), 2.4/2.5 (catalog caching + CDN for product
  images), 4.2 (rate limiting checkout/payment endpoints against abuse).

### 5.4 Social Media / News Feed (Twitter/Instagram-shaped)
- **Functional**: post content, follow/unfollow, home feed showing followed
  users' posts in roughly reverse-chronological or ranked order.
- **Non-functional**: read-heavy by an enormous margin, feed generation
  latency matters a lot, some staleness is fine.
- **Deep dive — the fan-out problem**: this is the module's centerpiece.
  - *Fan-out on write (push)* — when a user posts, immediately push the post
    into every follower's precomputed feed (cache, 2.4). Fast reads, but a
    celebrity with 50M followers makes a single post trigger 50M writes
    (the hot-key problem from 2.4, at write time instead of read time).
  - *Fan-out on read (pull)* — feed is assembled at read time by querying
    posts from everyone you follow. Cheap writes, but expensive, slow reads
    for users following many people.
  - *Hybrid* — push for normal users, pull (merged in at read time) for
    celebrity accounts — the standard real-world answer, and a good
    interview signal that you understand *when* each extreme breaks down.
- **Modules exercised**: 3.7 (sharding by user ID), 2.4 (precomputed feed
  cache), 2.7 (async fan-out via pub/sub on post creation), 4.1 (capacity
  math is what reveals the celebrity hotspot in the first place).

**→ leads to:** having built a few of these on paper, the remaining skill is
purely about *performance under interview conditions* — structuring the
45 minutes, communicating trade-offs out loud, and not getting lost in one
sub-problem. That's Module 6.

---

## Module 6 — System Design Interviews, Architecture, Trade-offs and Mock Interviews

### 6.1 A structured interview framework
Use the same skeleton every time (it's Module 5's five steps, timed):
1. **Clarify requirements** (2-5 min) — functional and non-functional. Ask,
   don't assume: read-heavy or write-heavy? Strong or eventual consistency
   acceptable? Rough scale (users, QPS)?
2. **Capacity estimate** (3-5 min) — 4.1's back-of-envelope math, stated out
   loud, because it drives every later decision (whether you need sharding
   at all, whether caching alone suffices).
3. **API design** (2-3 min) — the 3-6 endpoints that satisfy the
   requirements; keep it high-level.
4. **High-level architecture** (10-15 min) — draw the boxes (client → LB →
   service → cache → DB, async side-paths), narrating *why* each box exists
   as you add it (this is literally re-deriving Modules 1-4's causal chain
   live).
5. **Deep dive** (10-15 min) — the interviewer usually steers you to the
   hardest 1-2 sub-problems (Module 5's "deep dive" sections are exactly this).
6. **Wrap-up / trade-offs** (2-3 min) — explicitly name what you'd do
   differently at 10x scale, and what you sacrificed (usually consistency,
   cost, or complexity) to get what you gained.

### 6.2 Gathering requirements well
The single most common failure mode is skipping this and diving straight
into architecture. Ask about: read/write ratio, latency requirements (is
100ms fine, or does this need <10ms), consistency requirements (would a user
notice/care about a few seconds of staleness), and expected scale — because
the *same* feature (e.g. "a counter") is one line of code at 100 users and a
sharded, eventually-consistent, approximate-counting problem at 100M users.

### 6.3 Communicating trade-offs explicitly
The habit that separates "knows the vocabulary" from "can design a system":
narrate *which module concept* justifies each choice as you make it — e.g.
"I'm sharding by user ID (3.7) because a single Postgres instance (3.1) can't
take this write volume, and I'm adding a cache in front of the hot read path
(2.4) because the access pattern is read-heavy per the capacity estimate
(4.1)." Never present a decision as the only option — name the alternative
and why you didn't pick it.

### 6.4 Common trade-off axes — a cheat sheet
- **Consistency vs availability** (3.6) — under partition, do you reject or
  serve stale.
- **Latency vs consistency** (3.6's PACELC) — even without partition, do you
  wait for consensus or answer immediately from one replica.
- **Latency vs throughput** (4.1) — batching/queuing improves throughput at
  the cost of per-item latency.
- **Cost vs redundancy** (4.5) — every extra 9 of availability costs
  disproportionately more.
- **Read optimization vs write optimization** (3.4, 3.8) — indexes and
  denormalization speed reads and slow/complicate writes.
- **Simplicity vs flexibility** (3.3, 4.6) — a monolith with one Postgres
  instance is simpler to operate and reason about than microservices with
  sagas; only split when a concrete scaling or team-ownership problem
  demands it.

### 6.5 Mock interview practice
Do one full mock design per week (Module 5's list, roughly in increasing
difficulty: URL shortener → rate limiter → chat app → e-commerce → social
feed → design-a-distributed-cache/design-Redis-itself as a capstone), timed
at 35-45 minutes, ideally with another person interviewing you or recording
yourself and reviewing after. What interviewers actually score:
- Structured communication (did you follow 6.1's skeleton, or jump around).
- A working, coherent end-to-end design (not necessarily the "optimal" one).
- Depth on the deep-dive (can you go two levels below the box-and-arrow
  diagram when pushed).
- Explicit trade-off articulation (6.3) — this is graded as heavily as
  correctness at senior levels.

### 6.6 Common mistakes to eliminate in practice
- Skipping requirements gathering and designing the wrong system well.
- Going deep on one component (usually the database schema) for 20 minutes
  and running out of time before reaching the interesting distributed-systems
  parts.
- Silence while thinking — narrate your reasoning even when unsure; an
  interviewer scoring communication can't see a correct answer arrived at
  silently.
- Presenting a design with no acknowledged weaknesses — every real design
  has a trade-off; naming yours proactively reads as seniority, not weakness.
- Ignoring non-functional requirements after gathering them (e.g. stating
  "we need low latency" in step 1 and then never mentioning caching in step 4).

---

## The dependency chain, compressed

```
Module 1. Client-Server, HTTP, APIs, Databases (basics + auth)
   │
   ▼
Module 2. Load Balancers, Cache, CDN, Message Queues
   │            (scales single requests + decouples slow work)
   ▼
Module 3. SQL/NoSQL, Indexing, Replication, Sharding
   │            (the data layer these all sit in front of)
   ▼
Module 4. Scalability, Rate Limiting, High Availability
   │            (what happens under load, and when things break)
   ▼
Module 5. Real-world designs (URL Shortener, Chat, E-commerce, Social Media)
   │            (combine 1-4 into coherent, working systems)
   ▼
Module 6. Interviews: framework, trade-off articulation, mock practice
```
