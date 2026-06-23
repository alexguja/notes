# System Design Notes

Working notes on building large-scale systems: the primitives, the trade-offs, and —
most usefully — the **patterns** that recur across almost every design. The first half
covers the building blocks (networking, storage, indexing, caching, consistency,
concurrency, sharding). The second half is the part worth lingering on: the handful of
architectural recipes that solve the problems you actually hit at scale, each with the
non-obvious details that separate a sketch from a system that works.

One idea runs through all of it: **modern hardware is enormous, so start simple and add
machinery only when the numbers force you to.** A lot of "we need a distributed system"
problems fit on a single big box.

---

# Part I — Building blocks

## Scaling: vertical first, horizontal when you must

Two ways to handle more load: a bigger machine (**vertical**) or more machines
(**horizontal**). Horizontal gets the glory, but the moment you add a second node you
inherit the whole distributed-systems problem set — where work goes, where data lives,
how copies stay in sync, what happens when one node is slow. A single modern server goes
much further than intuition suggests, so reach for vertical first and go horizontal when
you genuinely outgrow one box, not reflexively.

Once you do spread out, two distribution problems appear:

- **Work distribution** — getting each request to a machine. A load balancer for
  synchronous traffic (plain round-robin is usually fine), a queue for async work. The
  goal is *even* load; one node at 90% while the rest idle at 10% means you gained almost
  nothing.
- **Data distribution** — getting each request the data it needs. Ideally a node answers
  from data it owns. When it must ask others (**fan-out**), keep the fan small. The
  pathological case is **scatter-gather** — one request fans out to many nodes and waits
  for all of them: network-heavy, fragile (any one failure stalls it), and bound by
  **tail latency** (you're only as fast as the slowest shard).

## Numbers that actually matter (2025)

Most over-engineering traces back to stale hardware intuitions.

| Component | One unit handles | Think about scaling when |
|---|---|---|
| **Cache** | ~1 ms reads, 100k+ ops/sec, up to ~1 TB RAM | dataset ~1 TB, 100k+ ops/sec, need <0.5 ms |
| **Database** | ~50k TPS, 5–30 ms disk reads (1–5 ms cached), 64 TiB+ | writes >10k TPS, uncached reads >5 ms, painful backups, geo |
| **App server** | 100k+ connections, 8–64 cores, 64–512 GB | CPU >70–80%, latency over budget, memory >80% |
| **Message broker** | ~1M msgs/sec, sub-5 ms, 50 TB, weeks of retention | ~800k msgs/sec, ~200k partitions, growing consumer lag |

Anchors: **same region ~1–2 ms, cross-region ~50–150 ms.** Memory is ~50× faster than a
cold disk read (1 ms vs ~50 ms). An *indexed* SSD row lookup is ~10 ms — fast enough that
a cache often buys nothing. CPU, not memory, is usually the first wall on app servers.
And a well-tuned Postgres does **20k+ simple writes/sec** — don't queue 5k writes/sec.
10M rows × 1 KB = **10 GB**; that's not a sharding problem. Do the arithmetic first.

## Networking and protocols

Three layers matter:

- **L3 (IP)** — routing/addressing, best-effort, no guarantees.
- **L4 (TCP/UDP)** — **TCP** is the reliable, ordered, connection-oriented workhorse
  (default for nearly everything). **UDP** is connectionless "spray and pray": no
  delivery or ordering guarantees, tiny header, fast — right when speed beats reliability
  and you can absorb loss (live video, gaming, VoIP, DNS). Browsers don't speak raw UDP
  except via WebRTC. (*QUIC*/HTTP3 ≈ "TCP done better" over UDP.)
- **L7** — the application protocols below.

**HTTP** is stateless (each request stands alone — which is exactly why HTTP services
scale trivially behind a load balancer). **HTTPS** adds TLS; encrypted ≠ trustworthy, so
never trust a user ID from a request body — derive identity from the auth token.

Building APIs on top:

- **REST** — model **resources** (nouns) acted on by verbs: `GET /users/{id}`,
  `POST /users`, `GET /users/{id}/posts`. Flexible, universal, the right default.
- **GraphQL** — client asks for exactly the fields it wants in one query, killing
  over/under-fetching. Shines with diverse clients (mobile) and fast-moving frontends;
  costs backend complexity.
- **gRPC** — RPC over HTTP/2 with Protocol Buffers (binary, schema'd): ~15 vs ~40 bytes,
  up to ~10× throughput, strong typing, streaming. The sweet spot is **internal
  service-to-service**; avoid for public APIs (binary, weak browser support). Common
  split: gRPC inside, REST at the edge.

**Load balancing** — **L4** (TCP/UDP, fast, protocol-agnostic) for **persistent
connections like WebSockets**; **L7** (understands HTTP, routes by path/header, can
terminate TLS) for ordinary request/response. Put one at the edge, assume the rest are
horizontally scaled.

## Storing data

The SQL-vs-NoSQL debate is mostly a trap — they overlap heavily. Pick by the concrete
feature you're relying on, not slogans.

- **Relational (Postgres, MySQL)** — tables, SQL, and three things you buy into: **joins**
  (powerful but a real bottleneck — minimize), **rich indexing**, and **transactions**
  (ACID, all-or-nothing). Default for structured, transactional data.
- **NoSQL (DynamoDB, Cassandra, MongoDB)** — key-value/document/column/graph, often
  schema-less, built to scale horizontally with tunable consistency. Cassandra in
  particular crushes **write-heavy** workloads via append-only (LSM) storage.

**Blob storage (S3, GCS)** — large unstructured objects don't belong in a database (kills
query/backup/replication performance). Put them in object storage (effectively infinite,
11 nines durable, ~$0.023/GB/mo) and keep a **pointer (URL) in the DB**. Rule of thumb:
over ~10 MB and not queried with SQL → blob storage. (The full upload/download machinery
is a pattern — see Part II.)

## Indexing

Without an index, finding a row means scanning every page off disk; an index is a
separate structure that points straight at the data, trading write cost and disk for fast
reads. Index in your primary database whenever you can.

| Index | Strengths | Use for |
|---|---|---|
| **B-tree** | sorted, balanced, exact + range, 2–3 disk reads | the default for almost everything |
| **LSM tree** | append-only, sequential batched writes | write-heavy: metrics, logs, Cassandra/RocksDB |
| **Hash** | O(1) exact match, no ranges | in-memory lookups (Redis) |
| **Geospatial** | proximity / "within N miles" | location data (Uber, Yelp) |
| **Inverted** | word → documents | full-text search (Elasticsearch/Lucene) |

- **B-tree** — nodes hold hundreds of keys, sized to a disk page (~8 KB), so a lookup in a
  millions-row table is 2–3 reads. Sorted, so equality *and* ranges *and* `ORDER BY` are
  all efficient; stays balanced under random writes. The Swiss-army knife.
- **LSM** — writes hit an in-memory memtable (+ write-ahead log), flushed in bulk to
  immutable **SSTables**; background **compaction** merges them. Many small random writes
  become a few big sequential ones — blazing writes, slower reads (a key may be in any of
  many SSTables; **bloom filters** and **sparse indexes** let reads skip most files).
- **Geospatial** — lat/long as two 1-D B-trees gives a rectangle, not a circle.
  **Geohash** encodes a 2-D point as a 1-D prefix string so an ordinary B-tree finds
  neighbors (what Redis geo uses); **quadtrees** recursively quarter space; **R-trees**
  (PostGIS default) use adaptive overlapping rectangles and handle shapes, not just points.
- **Inverted** — `LIKE '%x%'` can't use a B-tree, so it scans everything. Flip it: map
  each word → documents containing it. Real systems add tokenizing, lowercasing,
  stop-word removal, and **stemming** ("running"→"run") plus relevance scoring.

Two patterns: **composite index** `(user_id, created_at)` serves filter+sort in one
traversal, but only helps **left-prefixes** of its columns; **covering index** includes
the columns a query *reads* so it never touches the table (great for hot read paths, at a
storage cost).

## Consistency and CAP

Partitions are inevitable, so when one happens you choose **consistency** (every read
sees the latest write, refusing some nodes) or **availability** (always answer, possibly
stale). You can't have both during a partition.

**Default to availability.** Demand strong consistency only where a stale read causes real
damage — inventory (overselling), bookings (double-booking), money (balances).
Consistency is **per-feature, not per-system**: product descriptions can be eventually
consistent while inventory must be strong. "Consistency" is a spectrum (strong, eventual,
causal, read-your-writes) — pick the weakest level that's still correct.

## Concurrency primitives

When actors touch the same resource at once you get **race conditions**. Locking fixes
correctness but can wreck throughput, so lock **finely** (one row, not the table),
**briefly** (just the critical section), and **avoid it** when you can. The cheap-to-
expensive ladder of primitives:

1. **Transactions / atomicity** — `BEGIN … COMMIT` makes a group of ops all-or-nothing;
   solves a huge fraction of contention with no explicit locking.
2. **Optimistic concurrency** — assume conflicts are rare: do the work, then compare-and-
   swap on a version column, retry if it changed. No blocking; best for low contention.
3. **Pessimistic locking** — `SELECT … FOR UPDATE` up front. Predictable under genuinely
   high contention, at the cost of blocking.

The recipe for *cross-database* contention (2PC, sagas, reservations) is a pattern — Part II.

## Sharding, partitioning, consistent hashing

**Partitioning** splits data — *vertical* (different columns/features into different
stores) or *horizontal* (split rows). **Sharding** is horizontal partitioning across
separate database instances. It's a big step (cross-shard joins/transactions get hard),
so don't do it until storage, write throughput, backup windows, or geo demand it.

Everything hinges on the **shard key**, which must spread load evenly *and* keep related
data together. Strategies: **range-based** (contiguous ranges; simple, great range scans,
prone to hot shards — `created_at` sends all writes to the newest shard); **hash-based**
(`hash(key) % N`, even spread, but resharding remaps everything); **directory-based**
(lookup table, max flexibility, extra hop + SPOF, rarely worth it).

**Consistent hashing** fixes hash resharding and shows up everywhere (Cassandra, DynamoDB,
caches, LBs). Naive `% N` moves nearly every key when `N` changes. Instead, place nodes
*and* keys on a **ring**; a key belongs to the next node clockwise, so adding/removing a
node only moves its neighboring arc. **Virtual nodes** (each physical node at many ring
positions) smooth out lumpy distribution.

---

# Part II — Patterns

These are the recurring recipes. Each starts simple and escalates; the skill is stopping
at the simplest rung that meets the requirement.

## Scaling reads

Reads almost always outgrow writes — open Instagram and one screen fires 100+ reads
(photos, profiles, like counts) while you post once a day. Ratios start at 10:1 and reach
100:1+. Past a point this is physics, not code, so you climb a ladder:

1. **Optimize within the database first.** Add **indexes** on the columns you filter,
   join, and sort by (under-indexing kills far more apps than over-indexing). **Denormalize**
   for read-heavy paths — store redundant data so a page load is one single-table read
   instead of a four-table join; you pay on writes (update several places), which is fine
   when reads dominate. **Materialized views** precompute expensive aggregations (average
   rating, etc.) via a background job. Sometimes the answer really is just a bigger box.
2. **Scale horizontally** when you exceed roughly 50k–100k reads/sec on a well-indexed DB.
   **Read replicas** (leader handles writes, followers serve reads) add throughput *and*
   redundancy. The catch is **replication lag**: async replication is fast but a user may
   not see their own write immediately; sync replication is consistent but slower.
   **Functional/geographic sharding** also helps (US data in US databases).
3. **Add caching layers.** Access is wildly skewed — millions read the same viral tweet —
   so an in-memory cache (cache-aside in Redis) serves the hot set from RAM at sub-ms.
   CDNs push cacheable content to the edge (200 ms → <10 ms; can shed 90%+ of origin load),
   but only for data shared across users — never cache per-user data, there's no hit-rate
   benefit.

The deep stuff that makes or breaks read scaling:

- **Cache invalidation strategy** — TTL (simple, bounded staleness — drive it from a
  "data must be < N seconds stale" requirement), write-through (immediate, adds write
  latency), versioned keys (change the key on update so old entries become unreachable —
  no invalidation races, at the cost of one extra read for the version), or tagged
  invalidation. Different data tolerates different staleness; design around that.
- **Hot key, millions reading one entry** — a single Redis node can't serialize the same
  value 500k times/sec. **Request coalescing / single-flight** collapses concurrent misses
  so the backend sees *exactly N requests* (one per app server), no matter how many users
  ask. **Cache-key fanout** stores the value under k copies (`feed:taylor:0..k-1`); readers
  pick one at random, spreading load k-fold (costs memory + multi-key invalidation).
- **Cache stampede / thundering herd** — when a hot key with a TTL expires, every request
  misses *at the same instant* and hammers the DB (a self-inflicted DDoS). Fix with a
  rebuild lock (only the first request rebuilds, others wait — fragile if the rebuild is
  slow), **probabilistic early refresh** (as the entry ages, each request has a rising
  chance of refreshing it in the background, so refreshes spread over minutes instead of
  spiking), or a continuous background refresh for your most critical keys.

## Scaling writes

Writes are the harder half — you can't replicate your way out, and bursts with contention
are nasty. Four strategies, in order:

1. **Vertical scaling + write-optimized storage.** Confirm you're actually hitting disk
   I/O, CPU, or network limits (200-core boxes exist). Then pick storage that matches:
   **Cassandra** does 10k+ writes/sec vs ~1k for a relational DB on the same work, because
   its append-only commit log writes sequentially instead of seeking to update in place
   (you trade read performance for it). Time-series DBs, log-structured stores, and column
   stores make similar trades. Also: drop unneeded indexes, batch WAL flushes, disable
   expensive constraints during heavy write periods.
2. **Shard / partition.** If one server does 1k writes/sec, ten *should* do 10k — if the
   **partition key spreads writes evenly** (hash the userId/postId; *flat is good*).
   Sharding by something skewed like country overloads the China shard and idles New
   Zealand. Always ask "how many shards does this read hit?" — spreading writes perfectly
   but forcing every read to scatter across all shards just moves the bottleneck.
   **Vertical partitioning** is the complementary move: split a hammered `posts` table into
   `post_content` (write-once, read-many), `post_metrics` (high-frequency counter updates),
   and `post_analytics` (append-only time-series), each on storage tuned for its pattern.
3. **Absorb bursts with queues and load shedding.** Real traffic spikes (Black Friday 4×).
   Autoscaling is slow and databases hate scaling mid-fire, so either **buffer** writes in
   a queue (Kafka/SQS) so the DB drains at a steady rate — but a queue only survives
   *short* bursts; if inflow > processing rate indefinitely the backlog grows without
   bound and you've just hidden the real problem — or **shed load**: deliberately drop the
   least-valuable writes. Uber location pings arrive every few seconds, so dropping one is
   harmless — a fresher one is right behind it. Pick which writes matter and sacrifice the
   rest to stay alive.
4. **Batch and aggregate.** Change the *shape* of writes. **Batching** amortizes per-write
   overhead (a "like batcher" that tallies 100 likes in a 1-minute window into a single +100
   write — but verify the batching actually helps; batching posts that get 1 like/hour
   saves nothing). **Hierarchical aggregation** for extreme fan-in/fan-out (millions of live-
   stream viewers all liking comments): aggregate up through a tree of write processors
   (each owns a key range, merges over a window) and dis-aggregate down through broadcast
   nodes (assigned via consistent hashing), so no single node sees all N writes.

Deep dives: **resharding** without downtime uses **dual writes** — write to both old and
new shards, read-prefer the new, migrate gradually, then cut over. A **hot key too big for
one shard** (viral tweet, 100k likes/sec) gets **split into sub-keys** (`post1:0..k-1`) so
each shard takes a fraction; readers must check all sub-keys (read amplification is the
price). Works for aggregatable data (counts, likes), not for things that must stay atomic.

## Dealing with contention

The recipe for the race conditions above. **Golden rule: keep contended data in a single
database** — a local transaction is vastly simpler than anything distributed, and it's
achievable nine times out of ten. Within one DB, use pessimistic locking under high
contention and optimistic concurrency under low. When you genuinely must coordinate across
databases:

- **Two-phase commit (2PC)** — a coordinator runs *prepare* (everyone does the work but
  holds the transaction open) then *commit/abort*. Cross-system atomicity, but fragile and
  expensive: open transactions hold locks, a coordinator crash between phases strands
  participants, a slow node blocks everyone. The coordinator must persist its decision log
  first. Use sparingly.
- **Saga** — a sequence of *locally-committed* steps, each with a **compensating** action.
  Debit Alice (commit), credit Bob (commit); if crediting Bob fails, refund Alice. No
  long-held locks, no coordinator limbo — but the system is **temporarily inconsistent**
  mid-saga, so model "pending" states explicitly.
- **Reservations (distributed locks)** — often the cleanest fix is to *prevent* contention
  by creating an intermediate state. Ticketmaster moves a seat to "reserved" the instant
  you select it, shrinking the contention window from a 5-minute checkout to a millisecond.
  Uber sets a driver "pending_request"; carts put items "on hold." Back it with Redis+TTL
  (fast, Redis is a SPOF), a DB column (consistent, slower), or ZooKeeper/etcd (robust,
  consensus-backed). Always set an **expiry** so a crash doesn't deadlock the resource, and
  acquire multiple locks in a **consistent order** to avoid deadlocks.

The quiet hero across all of this is **idempotency**: if an operation carries an
idempotency key (or is naturally idempotent), retries and duplicate deliveries stop being
dangerous.

## Real-time updates

Pushing updates to users has two halves.

**Client-facing** — the protocol ladder, climbed only as far as needed:

- **Polling** — ask repeatedly. Simple, works everywhere, wasteful and laggy. Fine baseline.
- **Long polling** — server holds the request open until it has data, then the client
  re-asks. Near-real-time over ordinary HTTP infra — no special load balancers.
- **SSE** — one long-lived HTTP response the server streams many messages into. Efficient
  **one-way** push with built-in reconnect/replay. Gotchas: proxies close idle connections,
  and some networks buffer the whole stream and defeat it.
- **WebSockets** — persistent **bidirectional** TCP channel (upgraded from HTTP). Servers
  push unprompted; clients push back. The price is statefulness — every proxy/LB/firewall
  must support it, and millions of open connections force real accommodations. For chat,
  multiplayer, collaborative editing.
- **WebRTC** — direct **peer-to-peer** over UDP; peers find each other via a **signaling
  server** and punch through NATs with **STUN** (discover public address) + **TURN** (relay
  fallback). Hard; reserve it for audio/video.

**Server-side** — how a fleet of machines gets an event to the box holding the right
connection:

- **Pull** — clients/servers poll. Simplest, least efficient.
- **Consistent-hash ring of stateful servers** — route a user/room to a specific server
  that holds its state and connections. Good when per-connection processing is heavy
  (collaborative editing, Google Docs).
- **Pub/Sub** — publish events to a broker; whichever server holds the relevant connections
  subscribes and forwards. Decouples publisher from subscriber and handles fan-out cleanly
  (WhatsApp delivery).

## Handling large blobs

Routing a 2 GB upload *through* your app servers makes them dumb, slow, expensive pipes.
The pattern: your server stops touching bytes and becomes an **access-control ticket
booth** — it validates the request, hands out a temporary credential, and gets out of the
way.

- **Presigned URLs** — the server generates (entirely in memory, no call to storage) a
  signed, time-limited URL granting permission to PUT one specific object. The client
  uploads **directly to S3** at full regional speed. Bake restrictions into the signature
  (`content-length-range`, `content-type`) so a 5 MB image URL can't be used to upload a
  500 MB video. Downloads mirror this with signed URLs — validated by the storage service
  (S3) or, for CDN delivery, by edge servers using public-key crypto (CloudFront), no
  round trip to origin.
- **Resumable / multipart upload** — a 5 GB file over 100 Mbps is ~7 minutes; a drop at 99%
  shouldn't restart from zero. Split into parts (S3 multipart, 5 MB+), each independently
  uploaded and checksummed; on reconnect, query which parts landed and resume from the gap.
  Parts also give you a free progress bar. A final *complete* call assembles them; set a
  lifecycle rule to purge incomplete uploads after a day or two (they cost money).
- **State synchronization** — metadata lives in your DB, bytes live in S3, and they update
  at different times. Trusting the client's "done!" callback invites race conditions,
  orphaned files, and lies. Instead, let **storage emit an event** (S3 → SNS/SQS) on
  completion keyed by the same storage key you recorded, and update the row from that —
  removing the client from the trust equation. Add a **reconciliation job** to sweep up
  rows stuck "pending" when events are lost.
- **Fast downloads** — serve through a CDN (geography), and support **HTTP range requests**
  so large downloads are resumable and seekable (and adaptive-bitrate video works).

When *not* to: small files (<10 MB — the two-step dance isn't worth it), and anything
needing synchronous validation or compliance scanning *before* accepting bytes (proxy
those, in chunks).

## Long-running tasks (async worker pools)

Anything over a few seconds — video transcoding, PDF/report generation, bulk imports —
can't be synchronous: load balancers time out around 30–60 s, and a frozen spinner makes
users refresh and double-submit. Split the operation in two: the web server **validates,
writes a `pending` job record, pushes the job ID to a queue, and returns the ID in
milliseconds**; a separate **worker pool** pulls jobs, does the heavy lifting, stores
results, and updates status. You notify on completion (email, push, WebSocket).

Why it's worth the moving parts: **independent scaling** (cheap web boxes, GPU workers
for transcoding — sized separately), **fault isolation** (a crashed worker doesn't take
down the API; the job is retried), and far better UX. What you take on: eventual
consistency, status tracking, and a pile of failure modes — which is where the real
insight lives:

- **Worker crash mid-job** — a **heartbeat / visibility timeout** (SQS visibility timeout,
  RabbitMQ heartbeat, Kafka session timeout) detects the dead worker and re-queues the job.
  Tune the interval: too long delays recovery, too short marks live-but-paused workers
  (GC) as dead. ~10–30 s is a sane start.
- **A job that keeps failing** (bug, poison input) would retry forever and can take down
  the whole fleet as each worker chokes on it. After N failures (3–5), move it to a **dead-
  letter queue** for human inspection. A growing DLQ is a bug alarm.
- **Duplicate submissions** (impatient user clicks "Generate" 3×) — require an
  **idempotency key** (user + action + rounded timestamp); check for an existing job before
  creating one, and make the work itself idempotent so a retried job doesn't double-charge.
- **Backpressure** — when inflow outpaces workers the queue grows unbounded. **Autoscale on
  queue depth, not CPU** (by the time CPU is high the queue is already deep), and shed load
  by rejecting new jobs with "system busy" past a depth limit.
- **Mixed workloads** — a 5-second report stuck behind a 5-hour export is head-of-line
  blocking. **Separate queues by expected duration** (a "fast" queue with many small
  workers, a "slow" queue with few beefy ones), or chunk big jobs.
- **Job dependencies** (fetch → render → email) — for simple chains, each worker enqueues
  the next with full context; for branching/parallel flows, reach for a workflow engine ↓.

## Multi-step processes (workflows & durable execution)

When a process spans several flaky services and must survive crashes, retries, and waits
("charge payment → reserve inventory → ship → wait for a human → email"), hand-rolling it
gets ugly fast — you end up interleaving *business* logic (what if inventory's out?) with
*systems* logic (what if the server crashed between steps?) and scattering compensation
everywhere. The escalation:

1. **Single-server orchestration** — one service calls each step in sequence. Fine for
   simple cases, but a crash after charging payment and before reserving inventory leaves
   no memory of what happened; adding DB checkpoints and pub/sub callbacks turns into a
   hand-built, half-broken state machine.
2. **Event sourcing** — store the **sequence of events**, not current state, in a durable
   log (Kafka). Each worker consumes an event, does its work, and emits the next:
   `OrderPlaced` → payment worker → `PaymentCharged` → inventory worker → `InventoryReserved`.
   You get fault tolerance, scalability, a complete audit trail, and replayability — at the
   cost of building and debugging real infrastructure (why did no worker pick up this
   event? what lineage produced this state?).
3. **Workflow engines / durable execution** — what you actually want: describe the workflow
   once and let the engine handle persistence, retries, and recovery. **Temporal** (durable
   execution) lets you write the flow as ordinary-looking *code*; it records each **activity**
   result to a history DB, so after a crash another worker **replays** the workflow and
   skips already-completed activities. The catch that makes it work: **workflows must be
   deterministic** (fixed timestamps, seeded randomness — so replay makes identical
   decisions) and **activities must be idempotent** (they can be retried). **Signals** let a
   workflow wait days for an external event (a human signature) without consuming resources
   — no polling. **Managed/declarative** engines (AWS Step Functions, Airflow) express the
   same thing as JSON/DAGs — nicer visualization, less expressive.

Deep dives: **versioning** long-running workflows (run old and new code side by side, or
use a deterministic `patch()` so in-flight executions keep the legacy path); **state size**
(pass IDs not payloads; periodically recreate a long workflow from a checkpoint);
**exactly-once** is really *at-least-once + idempotent activity*.

Signal that you need this: "if step X fails we must undo step Y," or "all steps complete or
none do." Don't use it for CRUD, single-step async (a queue is enough), or latency-
sensitive synchronous calls.

## Proximity / geospatial services

"Find things near me" (Uber, Yelp, Gopuff) wants a **geospatial index** (above) plus
dividing the area into **regions** to prune the search space — instantly exclude
everywhere the user isn't. Worth the machinery only at hundreds of thousands to millions
of items; below that, scanning beats a specialized index. And proximity queries are almost
always **local** — users want what's near them, not a global search.

---

# Operational concerns

**Security** isn't a bolt-on. **AuthN/AuthZ** — delegate to a gateway/dedicated service;
derive identity from tokens, never request bodies. **Encryption** in transit (TLS) and at
rest; per-user keys mean a DB breach doesn't expose plaintext. **Rate-limit** endpoints —
many leaks come from scraping, not broken auth.

**Monitoring** in three layers: **infrastructure** (CPU/mem/disk/network — leading
indicators), **service** (latency, error rate, throughput), **application/business**
(active users, conversions — does it actually work for people).

# Recurring principles

- **Start simple; add complexity only when a requirement forces it.** Polling before
  WebSockets, one DB before sharding, synchronous before queues, vertical before horizontal.
- **Do the arithmetic before scaling.** Most "huge" datasets fit on one machine and most
  "high" write rates fit in one database.
- **Match consistency to the feature, not the system.**
- **Push state to the edges** (a broker or a database) so services stay stateless and
  horizontally scalable.
- **Make operations idempotent** — it turns retries and duplicate deliveries from hazards
  into non-events.
- **Watch tail latency and scatter-gather** — a request waiting on many nodes is only as
  fast, and as reliable, as the worst one.
- **A queue hides a capacity problem; it doesn't solve one.** Unbounded inflow means an
  unbounded backlog. Pair queues with backpressure or load shedding.
