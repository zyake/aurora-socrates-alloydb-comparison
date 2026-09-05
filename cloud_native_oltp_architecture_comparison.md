# Cloud-Native OLTP Architecture Comparison
## Aurora vs. Socrates / Azure SQL Hyperscale vs. AlloyDB

**Focus:** write path, WAL durability, page materialization, storage disaggregation, elasticity, durability/availability, cost, and the architectural lessons derived in this session.

**Scope note.** This report compares *architectural ideas*, not only commercial products. In particular, the 2025 *OLTP in the Cloud* paper explicitly distinguishes its modeled **Aurora-like** and **Socrates-like** architectures from the corresponding commercial services.

---

## Executive summary

The central lesson is that all three systems exploit the same fundamental WAL principle:

> A transaction does not need its modified data pages to be synchronously persisted before commit; it needs the relevant WAL/redo to reach a durable boundary.

The systems differ primarily in **where that durability boundary is located** and **how much additional work is coupled to it**.

| Architecture | Synchronous commit boundary | WAL / redo distribution | Redo / page materialization | Durable page capacity | Core philosophy |
|---|---|---|---|---|---|
| Traditional DB + remote block storage | WAL on remote block device | DB engine | DB engine | Remote SAN/EBS-like storage | Move conventional DB unchanged to cloud |
| **Aurora** | 4-of-6 protection-group storage quorum | Primary/storage protocol fans redo to six members | Aurora storage nodes | Aurora distributed storage | **Compute/storage disaggregation + smart storage** |
| **Socrates / Hyperscale** | **Landing Zone (LZ)** | **XLOG**, asynchronously after hardening | **Page Servers** | **XStore / backup storage** | **Separate durability, log distribution, and page service** |
| **AlloyDB** | **Low-latency regional log storage** | Regional WAL pipeline | **Log Processing Service (LPS)** | **Sharded regional block storage** | **Disaggregate even inside the storage layer** |

The most important architectural progression is therefore:

```text
Traditional DB
    |
    v
Compute + WAL + buffer manager + checkpoint
    |
    | remote block storage
    v
SAN / EBS-like storage


Aurora
    |
    v
Compute
    |
    | redo only
    v
Smart replicated storage
  - log durability
  - redo processing
  - page materialization
  - page serving
  - repair / recovery


Socrates
    |
    +---- sync WAL ----> Landing Zone
    |                    durability
    |
    +---- async WAL ---> XLOG
                         distribution / lifecycle
                              |
                              v
                         Page Servers
                         redo / hot pages
                              |
                              v
                            XStore
                         cheap capacity


AlloyDB
    |
    v
Regional Log Storage
    |
    | asynchronous WAL consumption
    v
Log Processing Service
    |
    v
Regional Block Storage
```

The strongest conclusion from this session is:

> **Aurora disaggregates compute from storage. Socrates and AlloyDB further disaggregate the storage subsystem according to function and NFR profile.**

This finer decomposition can improve resource elasticity and cost efficiency, but it increases distributed-system complexity and does **not** make replication, redo, or page writes disappear.

---

# 1. Research framing

The attached 2024 IEEE survey, *Cloud-Native Databases: A Survey*, identifies **compute-storage disaggregation** and **“log is the database”** among the key cloud-native database techniques, and surveys architectural choices for cloud-native OLTP systems.

That framing is useful, but the phrase “log is the database” must be interpreted carefully:

- It does **not** mean page images never need to be produced.
- It does **not** mean conventional databases synchronously flush dirty pages at every commit.
- It primarily means that the **durable log becomes the authoritative incremental representation of updates**, allowing page creation/recovery to be moved away from the transaction-processing compute node.

This distinction is essential for comparing Aurora, Socrates, and AlloyDB correctly.

---

# 2. Baseline: what a conventional WAL database already does

A conventional WAL/ARIES-style DBMS normally follows:

```text
UPDATE
  |
  +--> modify page in buffer pool
  |
  +--> generate WAL
          |
          v
     durable WAL
          |
          v
       COMMIT

       ... later ...

dirty buffer
   |
checkpoint / eviction
   |
   v
data-page write
```

Therefore:

> **Dirty-page persistence is generally not on the transaction commit critical path even in a traditional WAL database.**

For a cloud-hosted traditional database using a remote block device, however, both of these eventually cross the compute/storage network boundary:

```text
DB compute
   |
   +---- WAL -----------> remote block storage
   |
   +---- dirty pages ---> remote block storage
                          (later)
```

This creates two separate issues:

1. **Commit latency** depends on the durable WAL path.
2. **Aggregate network/storage traffic** includes both WAL and later page writes.

Cloud-native “log-based” designs attack primarily the second issue by moving page reconstruction across the compute/storage boundary.

---

# 3. Three different meanings of write amplification

A major lesson from the session is that the phrase **write amplification** is too ambiguous unless its layer is specified.

We should distinguish:

### 3.1 Compute-side write amplification

How much write work the DB compute process itself performs.

```text
Traditional:
    WAL generation
    + buffer management
    + dirty-page writeback/checkpoint I/O

Aurora/Socrates/AlloyDB:
    WAL generation
    + much less/no DB-compute page writeback to storage
```

### 3.2 Compute-to-storage network amplification

How many bytes cross the DB-compute/storage boundary.

For a small update to an 8 KiB page:

```text
Traditional remote-storage DB
    compact WAL
    +
    eventual 8 KiB page transfer

Log-shipping architecture
    compact WAL
    only from compute
```

### 3.3 System-wide physical amplification

What the whole service eventually performs:

- WAL replication
- WAL persistence
- redo replay
- page materialization
- multi-zone page/block replication
- checkpoints
- garbage collection
- backup/archive traffic

A design can dramatically reduce **compute-side** and **network-boundary** amplification while still doing substantial **system-wide** work.

That is exactly what happens in Aurora, Socrates, and AlloyDB.

---

# 4. Aurora architecture

## 4.1 Original design objective

The Aurora SIGMOD paper argues that after compute and storage are separated, the dominant bottleneck becomes the **network**. Aurora therefore sends **redo records instead of dirty pages** and pushes redo processing into a purpose-built distributed storage service.

Conceptually:

```text
Aurora writer
     |
     | redo
     v
+--------------------------------+
| Aurora smart storage           |
|                                |
|  replication                   |
|  redo persistence              |
|  redo application              |
|  page materialization          |
|  page serving                  |
|  repair / recovery             |
+--------------------------------+
```

This was a major architectural step beyond a conventional DB running on generic network block storage.

---

## 4.2 Protection groups and the write path

Aurora divides storage into small logical chunks/protection groups. A protection group is associated with six storage members across three Availability Zones.

```text
                    Aurora writer
                         |
                  page-specific redo
                         |
                         v
                protection group
                         |
       +---------+-------+-------+---------+---------+
       |         |       |       |         |         |
       v         v       v       v         v         v
      S1        S2      S3      S4        S5        S6
      AZ-A      AZ-A    AZ-B    AZ-B      AZ-C      AZ-C
       \         \       \       \         \         \
        +---------------- wait for 4 ----------------+
                         |
                         v
                    durable WAL
                         |
                         v
                       COMMIT
```

Important properties:

- **6 WAL/redo destinations**
- **4/6 write quorum**
- storage spread across **3 AZs**
- the failure model was explicitly designed around correlated AZ failure
- the database writer does **not** synchronously send dirty page images

Later Aurora storage refinements keep full materialized page images on only a subset of the six members; the 2025 cost paper models **three full page copies** plus the six-way log replication.

---

## 4.3 What Aurora eliminates

Aurora eliminates:

```text
DB compute --> dirty page --> remote storage
```

as the normal page persistence mechanism.

This reduces:

- DB-primary page-write I/O
- compute/storage network traffic for full pages
- foreground checkpoint pressure
- crash-recovery work on the primary

The storage service reconstructs page state from redo instead.

---

## 4.4 What Aurora does NOT eliminate

The updated page still has to exist eventually.

Storage-side work is roughly:

```text
redo receive
    |
redo persist
    |
sort / deduplicate / coalesce
    |
apply redo
    |
materialize page version
    |
persist / replicate page state
    |
GC / repair / backup
```

Therefore the correct statement is:

> **Aurora moves page-maintenance work from the database compute node to the storage service; it does not remove the work.**

This is a recurring lesson for all “log-is-data” architectures.

---

# 5. Aurora foreground overhead

A central hypothesis developed in this session is that Aurora’s primary write path has more topology awareness than a dedicated-log-service architecture.

Conceptually, for a redo record:

```text
page identifier
     |
     v
protection-group mapping
     |
     v
protection-group membership
     |
     v
six destination streams
     |
     v
ACK processing / 4-of-6 quorum
     |
     v
durable frontier
```

The **mapping operation itself** need not be expensive: protection-group membership can be cached and page-to-group computation can be deterministic.

The more significant costs are:

1. six-way network fan-out,
2. multiple concurrent destination queues/streams,
3. cross-AZ transmission,
4. ACK processing,
5. quorum-progress bookkeeping,
6. flow control and slow-member handling.

### Important evidence limitation

The 2025 *OLTP in the Cloud* analytical model explicitly accounts for Aurora’s approximately **6× WAL network amplification**, but does **not** model all fine-grained proprietary CPU/protocol overhead such as:

- per-record PG lookup CPU,
- six-stream queue management,
- ACK processing CPU,
- quorum bookkeeping CPU,
- storage-side redo CPU,
- proprietary packet/flow-control contention.

Therefore these are reasonable architectural concerns but **not directly measured causal explanations** for Aurora throughput limits.

---

# 6. Why local SSD makes sense for Aurora smart storage

Aurora’s storage nodes are not merely passive remote disks. They perform:

- redo ingestion
- redo processing
- page reconstruction
- page reads
- version maintenance
- repair

That makes low-latency local storage a natural implementation choice.

If every Aurora storage node itself sat on a separately replicated EBS-like block device, the system would conceptually stack replication:

```text
Aurora replication
     |
     +--> storage node A --> underlying block-storage replication
     |
     +--> storage node B --> underlying block-storage replication
     |
     +--> storage node C --> underlying block-storage replication
     ...
```

This could duplicate durability work and add another network-service boundary.

However, this does **not** imply that all disaggregated databases must use only local SSD. Socrates demonstrates a more tiered design:

```text
fast page-processing tier --> local SSD/cache
cheap durable tier        --> remote cloud storage
```

---

# 7. Socrates / Azure SQL Hyperscale architecture

Socrates decomposes the system much more aggressively.

```text
                         Primary
                            |
              +-------------+-------------+
              |                           |
          sync WAL                    async WAL
              |                           |
              v                           v
        Landing Zone                    XLOG
           (LZ)                           |
       fast durability                    |
              |                           v
              |                      Page Servers
              |                     redo / page cache
              |                           |
              |                           v
              +-----------------------> XStore
                                  durable / cheap capacity
```

The key conceptual decomposition is:

```text
durability != log dissemination != page materialization
```

---

# 8. Socrates Landing Zone (LZ)

## 8.1 Purpose

The LZ is optimized for:

- synchronous durable WAL append,
- very low commit latency,
- strong integrity/consistency,
- small working set,
- predictable write behavior.

The original Socrates implementation used Azure Premium Storage/XIO and stated that it maintained **three replicas**.

The paper characterizes LZ as:

- fast,
- durable,
- possibly expensive,
- intentionally small,
- organized as a circular buffer.

---

## 8.2 Commit path

The critical path is:

```text
transaction
    |
generate WAL block
    |
    v
Landing Zone
    |
durable write / write quorum
    |
    v
WAL hardened
    |
    v
COMMIT ACK
```

Most importantly:

> **Socrates waits for the LZ, not for XLOG, before acknowledging commit durability.**

---

# 9. Why XLOG exists if the LZ is already durable

LZ and XLOG operate on the same logical WAL but have very different workload/NFR characteristics.

| Dimension | LZ | XLOG |
|---|---|---|
| Primary role | Commit durability | Log distribution/lifecycle |
| Synchronization | **Synchronous** | **Asynchronous** |
| Critical latency | Extremely important | Not directly on commit path |
| Workload | Append-heavy | Fan-out/read/serve/filter |
| Capacity | Small | Larger working hierarchy |
| Reader count | Minimal | Potentially many consumers |
| Consumer lag | Not its main function | Must support lagging consumers |
| Ordering/gaps | Durable storage semantics | Reordering/gap handling |
| Cost model | Expensive is acceptable if small | Must scale economically |
| Long-term retention | No | Destage to XStore |

Combining both jobs into a single service would require one subsystem to simultaneously optimize for:

```text
very low tail-latency synchronous writes
+ many concurrent readers
+ filtering
+ consumer lag
+ cache hierarchy
+ long retention
+ cheap capacity
+ horizontal distribution
```

Socrates avoids that NFR conflict.

---

# 10. Parallel LZ and XLOG paths

The Primary sends log blocks to **both LZ and XLOG in parallel**:

```text
                         Primary
                            |
                    produce block B
                            |
               +------------+------------+
               |                         |
               v                         v
              LZ                    XLOG pending
          synchronous                 async
               |                         |
               |                         X
          B hardened                 not publishable
               |                         |
               +---- hardened notice ----+
                                         |
                                         v
                                  LogBroker / consumers
```

This overlaps durability latency and dissemination latency.

Without overlap:

```text
LZ durable
   |
then send XLOG
```

Downstream propagation latency would roughly accumulate both steps.

With overlap, XLOG can already possess the bytes when the LZ durability barrier completes.

---

# 11. Why XLOG cannot immediately publish received WAL

If XLOG were to forward an unhardened WAL record to a Page Server/Secondary and the Primary then failed before that record became durable in LZ, the system could produce an impossible history:

```text
Durable history:
100 -> 101 -> 102

Consumer history:
100 -> 101 -> 102 -> 103
                    ^
                    not durable
```

The Socrates paper calls this unsafe idea **speculative logging**.

Therefore XLOG follows an invariant conceptually equivalent to:

```text
received(B) AND hardened(B) AND ordering-valid(B)
            |
            v
       publishable(B)
```

The Primary informs XLOG which blocks have become durable in the LZ. XLOG then moves them from its **pending area** to the **LogBroker**, while also filling gaps and restoring order if necessary.

This is effectively a distributed version of the WAL ordering principle:

> **Consumers must not make non-durable history externally meaningful.**

---

# 12. XLOG storage hierarchy and destaging

XLOG also turns a small expensive durability tier into an economical log hierarchy:

```text
             recent WAL
                 |
                 v
                LZ
          fast / expensive
                 |
              XLOG
            destaging
           /         \
          v           v
    local SSD       XStore
      cache       cheap long-term
```

This allows:

- low commit latency,
- fast access to recent WAL,
- long retention at lower cost,
- Point-in-Time Recovery support,
- recycling of the finite LZ circular buffer.

The Socrates paper states that if LZ fills with log that has not been destaged, update processing eventually has to stop. Therefore XLOG is **not on the immediate transaction critical path**, but it is necessary for **sustained write availability**.

---

# 13. Socrates Page Servers

Page Servers perform the work that has not disappeared merely because the Primary writes only WAL.

```text
XLOG
  |
  | filtered WAL
  v
Page Server
  |
  +--> replay WAL / redo
  +--> maintain page state
  +--> serve page reads
  +--> local memory/SSD cache
  +--> checkpoint modified pages
           |
           v
         XStore
```

Therefore:

> **Socrates still performs dirty-page/checkpoint work; it moves that work away from the Primary and makes it asynchronous with respect to transaction commit.**

This distinction is critical.

---

# 14. Socrates NFR decomposition

Socrates’ components can be interpreted as NFR-specialized tiers.

```text
LZ
    latency + synchronous durability

XLOG
    throughput + fan-out + ordering + lifecycle

Page Server
    redo CPU + random page access + fast cache

XStore
    capacity + durability + low cost

Primary
    transaction execution
```

This supports a general architectural principle:

> **Separate components not only by function, but by conflicting NFR profiles.**

---

# 15. Socrates performance evidence

The original Socrates SIGMOD paper included a write-heavy logging experiment.

With a CDB workload designed to saturate logging:

| Architecture | Log throughput | CPU |
|---|---:|---:|
| Previous SQL HADR | **56.9 MB/s** | 46.2% |
| Socrates | **89.8 MB/s** | 73.2% |

That is approximately a **58% increase in logging throughput**.

The paper attributes the difference partly to the fact that the older HADR architecture drove log/database backup work from the compute nodes, while Socrates pushed backup/storage responsibilities into lower storage tiers.

### Critical caveat

This benchmark is:

```text
Socrates vs. previous Azure SQL HADR
```

not:

```text
Socrates vs. Aurora
```

Therefore it demonstrates the benefit of Socrates’ decomposition relative to the older Microsoft architecture, but it does **not prove** that Hyperscale universally writes faster than Aurora.

---

# 16. Aurora vs. Socrates: foreground write path

The contrast can be summarized as follows.

## Aurora

```text
generate redo
     |
     v
identify protection group
     |
     v
use six-member topology
     |
     v
fan out redo
     |
     v
track 4/6 durability
     |
     v
COMMIT
```

## Socrates

```text
generate WAL
     |
     v
one logical durable-write interface
     |
     v
Landing Zone durability
     |
     v
COMMIT

         meanwhile/asynchronously
                    |
                    v
                  XLOG
                    |
                    v
              Page Servers
```

Thus several Aurora concerns are **removed from the Socrates Primary**, but not necessarily from the entire system.

| Concern | Aurora Primary / foreground protocol | Socrates Primary | System-wide |
|---|---|---|---|
| Page → storage-shard mapping | Yes | Not for commit durability | Page partitioning still exists |
| Replica membership awareness | More direct | Largely abstracted | Still exists below services |
| Six storage streams | Yes | No | LZ itself is replicated |
| Quorum tracking | Aurora storage protocol | Delegated to LZ | Still physically required |
| Page Server dissemination | Coupled to storage design | XLOG async | Still required |

The precise formulation is:

> **Socrates eliminates or delegates these concerns from the Primary commit path; it does not abolish distributed storage complexity.**

---

# 17. Socrates cost / elasticity argument

The 2025 *OLTP in the Cloud: Architectures, Tradeoffs, and Cost* paper calls Socrates-like **even more disaggregated** than Aurora-like because it separates log service from page service.

Its model finds that for very high transaction rates on small datasets, **Socrates-like can be cost-optimal**, and in one modeled region Aurora-like was about **70% more expensive**, because Aurora-like could not independently scale page and log service resources.

The model also describes Socrates-like as a reasonable default choice across a broad set of moderate-durability workloads.

This supports the key resource-coupling argument:

```text
Write-heavy workload needs:
    more WAL bandwidth
    more redo CPU

but may NOT need:
    more database capacity

Socrates:
    scale log/page processing separately

Aurora-like:
    log/page/storage resources are more coupled
```

---

# 18. Aurora durability and cost trade-off

The same 2025 paper gives Aurora-like a major advantage in durability.

Because of six-way log replication, it models Aurora-like at roughly **20 “nines” of durability** under its assumptions.

At the same time, it highlights the cost of replicated database state:

> For large, mostly cold datasets, the threefold full-database replication can make Aurora-like more than twice as expensive as alternatives in the model.

This leads to a useful customer-value formulation:

```text
Customer first needs:

durability >= business requirement
availability >= business requirement

Once those constraints are satisfied:

cost
latency
throughput
elasticity
often become the dominant differentiators.
```

Thus, extreme additional durability can have diminishing customer value if the alternative architecture already comfortably meets the application’s durability requirement.

---

# 19. Real Aurora benchmark evidence and its limitation

The 2025 cost paper also validated its model against commercial Aurora PostgreSQL.

It reports that in workloads containing updates it could not obtain more than about **0.5 million tx/s**, despite the analytical model predicting more in several cases.

For one 1-TB case, the model predicted roughly 1.1 GB/s of page reads, while the actual system reached about 630 MB/s despite available network bandwidth and moderate CPU utilization.

The authors say they **suspect contention and I/O-stack overhead**, but they cannot profile Aurora internally and therefore do not know the exact cause.

This is important for our discussion:

### The paper proves

- real update-heavy Aurora behavior deviated from the simplified resource model;
- a scalability limit/overhead appeared in those experiments.

### The paper does NOT prove

- protection-group routing caused it;
- 4/6 ACK bookkeeping caused it;
- storage-side redo processing caused it;
- six-way fan-out alone caused it.

Those remain plausible hypotheses requiring deeper instrumentation or internal AWS data.

---

# 20. AlloyDB architecture

AlloyDB applies an architecture that is conceptually much closer to Socrates than to the original Aurora design.

Google describes three storage-layer components:

1. **Low-latency regional log storage**
2. **Log Processing Service (LPS)**
3. **Failure-tolerant sharded regional block storage**

```text
                       PostgreSQL Primary
                              |
                         generate WAL
                              |
                              v
                 +-----------------------+
                 | Regional Log Storage  |
                 | low-latency durability|
                 +-----------+-----------+
                             |
                         async WAL
                             |
                             v
                 +-----------------------+
                 | Log Processing Service|
                 | LPS                   |
                 |                       |
                 | redo                  |
                 | block materialization |
                 | page/block serving    |
                 +-----------+-----------+
                             |
                             v
                 +-----------------------+
                 | Regional Block Storage|
                 | sharded + durable     |
                 +-----------------------+
```

Google explicitly states that the storage service itself further disaggregates compute and storage, so that **block storage can scale separately from log processing**.

---

# 21. AlloyDB commit path

The write lifecycle is:

```text
SQL UPDATE/INSERT/DELETE
        |
        v
Primary modifies in-memory state
        |
        v
generate PostgreSQL WAL
        |
        v
synchronously persist WAL
to regional log storage
        |
        v
transaction durable / commit
        |
        | asynchronous
        v
LPS consumes WAL
        |
        v
redo + block materialization
        |
        v
regional block storage
```

Thus:

```text
COMMIT requires:
    durable WAL

COMMIT does NOT require:
    LPS redo completion
    block materialization completion
```

This is the same fundamental separation we identified in Socrates.

---

# 22. AlloyDB LPS elasticity

A particularly advanced aspect of AlloyDB is that the **redo/page-processing tier itself is horizontally elastic**.

Google describes:

- persistence partitioned into block shards,
- each shard assigned to an LPS,
- multiple shards per LPS possible,
- dynamic shard-to-LPS reassignment,
- adding/removing LPS resources without copying durable data.

```text
                 regional WAL
                      |
           +----------+----------+
           |          |          |
           v          v          v
         LPS-1      LPS-2      LPS-3
           |          |          |
        shard A    shard B     shard C
           \          |          /
            +---------+---------+
                      |
                      v
              regional block store
```

This directly addresses one of the coupling concerns discussed for Aurora:

```text
Need more redo CPU?
    --> add/rebalance LPS capacity

Need more durable capacity?
    --> scale block storage

Need more SQL CPU?
    --> scale database compute
```

The resources do not need to grow in lockstep.

---

# 23. AlloyDB still has substantial replication/write work

Disaggregation does not eliminate multi-zone durability costs.

Google states that:

- storage is distributed across **three zones**,
- each zone has a **complete copy of database state**,
- each log record is processed in each zone,
- blocks are synchronously replicated across zones in the storage layer.

Conceptually:

```text
                   Regional WAL
                       |
          +------------+------------+
          |            |            |
          v            v            v
        Zone A       Zone B       Zone C
          |            |            |
         LPS          LPS          LPS
          |            |            |
          v            v            v
      DB state A   DB state B   DB state C
       complete     complete     complete
```

Therefore AlloyDB is not “low write amplification” in an absolute system-wide sense.

Its advantage is better stated as:

> **Replication and redo work are placed behind independently scalable services and kept off the latency-sensitive database-primary path.**

---

# 24. Socrates vs. AlloyDB

The mapping is striking:

| Socrates | AlloyDB | Role |
|---|---|---|
| Primary | PostgreSQL Primary | Transaction execution |
| **LZ** | **Regional log storage** | Synchronous WAL durability |
| **XLOG** | Partly log pipeline/service functions | WAL dissemination/lifecycle |
| **Page Server** | **LPS** | Redo + page/block materialization |
| **XStore** | **Regional block storage** | Durable database blocks/capacity |
| RBPEX/local SSD | Local caches/LPS caches | Fast working set |

The key difference is that Socrates exposes a distinct XLOG dissemination/lifecycle layer in the published architecture, whereas AlloyDB’s public architecture is presented more directly as:

```text
Regional Log Store -> LPS -> Regional Block Store
```

AlloyDB also emphasizes independent scaling of LPS resources and dynamic shard-to-LPS mapping.

---

# 25. Architectural generations

A useful historical taxonomy is:

## Generation 0 — traditional cloud-hosted DB

```text
DB engine
  + WAL
  + buffer pool
  + checkpoint
       |
       v
generic remote block storage
```

**Strength:** simple migration model  
**Weakness:** generic storage/network path retains monolithic DB behavior.

---

## Generation 1 — Aurora-style smart storage

```text
DB compute
    |
    | redo only
    v
smart distributed storage
```

**Innovation:** move page maintenance/recovery across network boundary.

**Strengths:**

- very strong fault tolerance,
- reduced page traffic from DB compute,
- fast recovery,
- self-healing storage.

**Trade-offs:**

- log and page functions remain comparatively coupled,
- six-way WAL fan-out,
- multiple full data copies,
- more topology awareness in write/storage protocol.

---

## Generation 2 — Socrates-style functional disaggregation

```text
Compute
  |
  +--> fast durable log tier
  |
  +--> async log distribution
             |
             v
        page-processing tier
             |
             v
        cheap durable tier
```

**Innovation:** separate storage functions according to distinct NFRs.

**Strengths:**

- simpler Primary durability abstraction,
- page work removed from commit path,
- cheaper durable capacity,
- independent resource sizing.

**Trade-offs:**

- more distributed components,
- more progress watermarks/leases/backpressure,
- XLOG/Page Server failure management,
- asynchronous backlog complexity.

---

## Generation 2+ — AlloyDB multi-layer storage disaggregation

```text
Compute
  |
  v
Regional Log Store
  |
  v
elastic LPS
  |
  v
Regional Block Store
```

**Innovation:** independently elastic redo-processing compute inside the storage layer.

**Strengths:**

- WAL append optimized separately,
- redo CPU scales independently,
- block capacity scales independently,
- database compute need not know storage topology.

**Trade-offs:**

- three-zone complete database state remains expensive,
- sophisticated distributed block/redo coordination,
- proprietary internals limit external analysis.

---

# 26. Why Aurora did not start with the Socrates/AlloyDB architecture

There is no public AWS paper saying “we considered Socrates and rejected it,” so this section distinguishes evidence from inference.

## Supported historical facts

Aurora’s original papers focused strongly on:

- network becoming the bottleneck,
- reducing compute-to-storage traffic,
- pushing redo into smart storage,
- surviving correlated AZ failures,
- fast recovery/self-healing.

Socrates was published later (2019), and AlloyDB later still (2022).

## Reasonable architectural inference

Aurora solved the immediate cloud problem:

```text
"How do we remove dirty-page/network/recovery pressure
 from a conventional DB compute node?"
```

Socrates asked the next question:

```text
"After moving storage work out of compute,
 why should durability, log distribution,
 redo, page serving, and cheap capacity
 still be one storage architecture?"
```

AlloyDB further refined that idea.

Therefore it is reasonable to call Socrates/AlloyDB **later-generation disaggregation**, but not correct to say Aurora was a poor design for its original goals.

---

# 27. Why Aurora has not simply replaced its storage architecture

This is necessarily more inferential.

Replacing:

```text
Primary -> Aurora Storage
```

with:

```text
Primary -> Log Service -> Redo Service -> Page Store
```

would be close to introducing a new storage engine, with enormous compatibility and correctness risk.

It would require revalidation of:

- crash consistency,
- failover,
- protection-group recovery,
- backup/PITR,
- Global Database behavior,
- encryption,
- repair,
- version upgrades,
- engine-specific redo semantics,
- existing customer fleets.

AWS instead appears to have evolved around the established storage architecture, e.g. storage refinements and higher-level scale-out mechanisms, rather than publicly replacing the core six-member protection-group model.

This is a plausible product/engineering strategy, not a proven internal AWS decision rationale.

---

# 28. Durability vs. availability: do not conflate them

A recurring correction in this session was to separate:

### Durability

> Will acknowledged committed data still exist after failures?

### Availability

> Can the service continue serving requests during/after failures?

Aurora’s 6-way storage quorum gives it exceptionally strong storage durability/fault tolerance.

Socrates deliberately **separates durability and availability**:

- LZ/XStore: durable truth
- Compute/Page Servers: availability/performance
- XLOG: dissemination and log lifecycle

AlloyDB similarly separates WAL durability, redo compute, and replicated block state.

Therefore:

> “Aurora has stronger durability” is much better supported than the blanket statement “Aurora is more available than Socrates.”

---

# 29. Infrastructure / AZ hypothesis

Another hypothesis from this session was that Aurora/Socrates differences may originate partly in AWS vs. Azure infrastructure history.

The strong form:

> “AWS AZs are real datacenters, Azure AZs are not”

is incorrect today. Both providers define availability zones as physically isolated datacenter locations/groups with independent infrastructure.

A more defensible historical hypothesis is:

```text
AWS
  early first-class AZ failure boundary
       |
       v
Aurora explicitly internalizes
3-AZ durability topology


Azure
  historically strong regional storage/fault-domain primitives
       |
       v
Socrates delegates durability
to cloud-storage services
```

Aurora’s original papers explicitly derive the six-member design from an AZ-correlated failure model.

Socrates explicitly builds on Azure Premium Storage and XStore.

However:

> **No source reviewed in this session proves that the datacenter topology directly caused the database architecture.**

Treat that causal explanation as a research hypothesis, not an established fact.

---

# 30. Comprehensive comparison matrix

| Dimension | Traditional + RBD/SAN | Aurora | Socrates / Hyperscale | AlloyDB |
|---|---|---|---|---|
| DB compute writes dirty pages to remote storage | Yes | **No** | **No** | **No** |
| WAL needed synchronously at commit | Yes | Yes | Yes | Yes |
| Synchronous durability target | Remote block WAL | 4/6 storage PG | **LZ** | **Regional log store** |
| WAL fan-out visible to DB primary | Low/logical device | **High: 6 members** | **Low: logical LZ** | **Low: logical regional log service** |
| Page placement involved in commit durability | Storage device implicitly | **Protection-group specific** | **No** | **No** |
| Redo application location | DB compute | Storage node | Page Server | LPS |
| Page materialization | DB compute | Storage node | Page Server | LPS |
| Checkpoint/page write work disappears? | No | No; transformed | **No; Page Server does it** | No; LPS/block tier does it |
| Fast local SSD tier | DB/host dependent | Storage nodes | Page Servers / RBPEX | caches / LPS |
| Cheap remote capacity tier | RBD itself | Less separated | **XStore** | **Regional block storage** |
| Separate log distribution tier | No | No | **XLOG** | Not exposed as a distinct analogous tier |
| Independent redo compute scaling | No | Limited/coupled | Better via Page Server scale-out | **Strong: elastic LPS** |
| Multi-zone page state | Storage-dependent | ~3 full copies in modeled architecture | Page copies + durable backup model | **Complete state in each of 3 zones** |
| Primary foreground topology awareness | Low | **Higher** | **Low** | **Low** |
| Architectural component count | Low | Medium | **High** | High |
| Extreme durability emphasis | Storage-dependent | **Very high** | Strong, but different model | Strong multi-zone |
| Functional/NFR decomposition | Low | Medium | **Very high** | **Very high** |
| Best conceptual strength | Familiarity | Durability + smart storage | Cost/elasticity decomposition | Elastic storage-layer processing |
| Main conceptual weakness | Network/I/O coupling | Storage-function coupling | Distributed-service complexity | Complex multi-zone replicated pipeline |

---

# 31. Customer-oriented interpretation

Customers usually do not optimize “maximum durability” independently.

A more realistic objective is:

```text
minimize:
    cost
    latency

maximize:
    throughput
    elasticity

subject to:
    durability >= requirement
    availability >= requirement
```

This matters because once an alternative system already satisfies the required durability/availability, additional redundancy may have diminishing customer value.

### Workload tendencies

| Workload | Architecturally attractive direction |
|---|---|
| Extreme durability/RPO sensitivity | Aurora-like redundancy may be attractive |
| High-write, small/medium DB | Fine log/page separation can be attractive |
| Large mostly-cold DB | Cheap durable capacity + smaller fast tier |
| Highly variable write load | Independently scalable log/redo tiers |
| Read-scale workload | Shared storage + cheap read compute |
| Development/test | Lower cost may dominate extreme durability |

This does not prove any one commercial product is always superior; it identifies the workload dimensions that should drive evaluation.

---

# 32. What “cheaper and faster” evidence really says

The strongest academic/model evidence in this session is:

1. **Socrates paper**
   - Real Socrates vs previous Azure HADR
   - Write-heavy logging: 89.8 MB/s vs 56.9 MB/s
   - Demonstrates benefit of moving storage/backup work away from compute

2. **2025 OLTP cost paper**
   - Models Aurora-like and Socrates-like architectures
   - Socrates-like is often cost-optimal/nearly cost-optimal under moderate durability
   - For very high transaction rates on small datasets, Aurora-like can be ~70% more expensive in the model
   - Aurora-like threefold DB replication can dominate cost for large cold data
   - Aurora-like modeled durability is exceptionally high
   - Real Aurora update workloads exposed an unexplained scaling gap

3. **AlloyDB official architecture**
   - Google explicitly motivates further storage-layer disaggregation for elasticity
   - LPS can scale independently
   - WAL durability is separated from redo/page processing

### What is still missing

There is no clean peer-reviewed experiment of:

```text
same DB engine
same transaction semantics
same hardware budget
same durability target
same availability target

Aurora-style storage
        vs
Socrates-style storage
        vs
AlloyDB-style storage
```

Therefore product-level “X is faster than Y because of architecture” claims must remain cautious.

---

# 33. Key session conclusions: claim audit

| Claim | Assessment |
|---|---|
| Traditional WAL DB does not normally sync dirty pages at commit | **Correct** |
| Log-based cloud DB eliminates dirty-page work | **Incorrect** |
| It eliminates compute-side dirty-page network writes | **Correct** |
| Storage/page nodes still materialize/checkpoint pages | **Correct** |
| Aurora uses redo-only compute→storage writes | **Supported** |
| Aurora’s six-way redo replication imposes real network amplification | **Supported** |
| The page→PG lookup itself is necessarily very expensive | **Not established** |
| Six-stream fan-out/quorum handling can add foreground overhead | **Architecturally plausible; detailed CPU cost not publicly quantified** |
| Socrates waits for XLOG at commit | **Incorrect** |
| Socrates waits for LZ durability at commit | **Correct** |
| XLOG receives WAL asynchronously and only publishes hardened WAL | **Correct** |
| LZ and XLOG exist because they have different NFR/workload profiles | **Strongly supported interpretation** |
| Socrates Page Servers still checkpoint/write pages | **Correct** |
| Socrates eliminates replication/quorum work system-wide | **Incorrect** |
| Socrates removes/delegates much of it from the Primary | **Correct** |
| Socrates is more functionally disaggregated than Aurora | **Supported by 2025 paper** |
| Socrates is always faster than Aurora for writes | **Not proven** |
| Socrates-like is often more cost efficient at moderate durability | **Supported by model** |
| Aurora has exceptionally high durability in the model | **Supported** |
| Aurora’s 3× full DB replication can be costly for cold datasets | **Supported** |
| AlloyDB resembles Socrates in commit/page separation | **Strongly supported** |
| AlloyDB eliminates multi-zone write amplification | **Incorrect** |
| AlloyDB makes redo processing independently elastic | **Supported** |
| Azure AZs are not real physical DC failure domains | **Incorrect today** |
| Cloud infrastructure history may have influenced DB architecture | **Plausible hypothesis, not proven causally** |

---

# 34. The deepest architectural lesson

The evolution can be understood as progressively shrinking the amount of work on the synchronous transaction path.

```text
Traditional cloud DB
--------------------
commit:
    durable WAL

background on same DB instance:
    page management
    checkpoint
    page writes


Aurora
------
commit:
    partition-aware redo
    six-way storage fan-out
    quorum durability

background storage:
    redo
    pages


Socrates
--------
commit:
    one logical LZ durable append

background:
    XLOG distribution
    Page Server redo
    checkpoint
    XStore


AlloyDB
-------
commit:
    regional durable WAL append

background:
    elastic LPS redo
    regional block materialization
```

A useful design metric is therefore:

> **How much physical page placement, replica topology, redo processing, and consumer dissemination is visible on the transaction commit path?**

By that metric:

- Traditional RBD: page work is largely outside commit but remains on DB compute.
- Aurora: page writes are gone from compute, but the redo commit protocol is storage-shard/quorum aware.
- Socrates: page placement/dissemination is mostly outside the commit durability boundary.
- AlloyDB: database compute commits against a regional log abstraction; redo/block placement is behind elastic storage services.

---

# 35. A second deep lesson: “move” vs. “eliminate”

For every cloud-native optimization, ask:

```text
What work disappeared?
What work merely moved?
What work became asynchronous?
What work became independently scalable?
What work became cheaper because it moved to a different tier?
```

For example:

| Operation | Aurora | Socrates | AlloyDB |
|---|---|---|---|
| Dirty-page network traffic from DB primary | Eliminated | Eliminated | Eliminated |
| Page materialization | **Moved** to storage | **Moved** to Page Server | **Moved** to LPS |
| WAL replication | Still required | Still required below LZ | Still required below regional log/storage |
| Checkpoint-like work | Transformed/storage-managed | Page Server → XStore | Storage-layer materialization |
| Multi-zone redundancy | Explicit protection-group scheme | Underlying services / architecture | Explicit 3-zone block state |
| Foreground complexity | Reduced vs conventional page I/O, but quorum-aware | Strongly reduced at Primary | Strongly reduced at Primary |

This vocabulary prevents misleading statements such as “log-based systems eliminate write amplification.”

---

# 36. Third deep lesson: specialization by NFR

The Socrates and AlloyDB architectures strongly suggest the following design principle:

```text
Do not force one storage tier to optimize:

  synchronous WAL latency
  + redo CPU
  + random page serving
  + cheap capacity
  + long retention
  + high fan-out
  + multi-zone durability
```

Instead specialize:

```text
fast append tier
    |
redo / distribution compute
    |
hot page/cache tier
    |
cheap durable capacity tier
```

This is analogous to a memory/storage hierarchy, but applied to **distributed database services and their NFRs**.

---

# 37. Fourth deep lesson: durability has a price-performance frontier

Aurora demonstrates that aggressive replication can produce extraordinary durability.

Socrates/AlloyDB demonstrate that other decomposition points can offer:

- lower coupling,
- cheaper capacity,
- better workload-specific scaling.

The correct design question is therefore not:

> “Which database has the most durability?”

It is:

> **“What is the cheapest architecture that satisfies the required durability, availability, latency, and throughput simultaneously?”**

That is exactly the multi-dimensional framing used by the 2025 *OLTP in the Cloud* paper.

---

# 38. Open research questions

The session exposed several areas where the public literature is still incomplete.

## 38.1 Aurora foreground protocol cost

Quantify:

```text
CPU(page -> PG mapping)
CPU(destination dispatch)
CPU(ACK processing)
CPU(quorum frontier)
PPS overhead
cross-AZ serialization cost
```

and determine how much each contributes under write-heavy loads.

---

## 38.2 Storage-side redo cost

Compare system-wide:

```text
logical WAL byte
   ->
network bytes
   ->
durable log bytes
   ->
redo CPU cycles
   ->
materialized block bytes
   ->
replicated page bytes
   ->
GC/backup bytes
```

for Aurora, Socrates, and AlloyDB.

This would be much more informative than simply comparing logical TPS.

---

## 38.3 Durability-normalized price/performance

A fair benchmark should normalize:

```text
same RPO
same durability probability
same AZ-failure guarantee
same number of readable replicas
same backup retention
same DB size
same engine semantics if possible
```

Otherwise a cheaper architecture may simply be providing a weaker failure guarantee.

---

## 38.4 Cloud infrastructure → DB architecture causality

A useful research framework would trace:

```text
physical DC topology
        |
failure-domain abstraction
        |
cloud storage primitives
        |
database durability boundary
        |
replication topology
        |
write amplification
        |
cost / latency / elasticity
```

across AWS, Azure, and Google Cloud.

This causal chain is intuitively compelling but not yet well established by the papers reviewed in this session.

---

# 39. Recommended evaluation framework

When comparing a new cloud database, fill out this matrix rather than only labeling it “NoSQL,” “NewSQL,” or “disaggregated.”

## A. Commit path

- What exact component is the synchronous durability boundary?
- How many logical network destinations does the DB primary contact?
- How many physical replicas are required before ACK?
- Does the DB primary understand replica topology?
- Does commit depend on page ownership/sharding?

## B. Log path

- Who persists WAL?
- Who distributes WAL?
- Is distribution synchronous or asynchronous?
- Is WAL filtering performed?
- How are lagging consumers handled?
- How long is WAL retained in expensive vs. cheap storage?

## C. Page path

- Who executes redo?
- Who materializes pages?
- Where are hot pages cached?
- Who checkpoints them?
- Is page service independently scalable?

## D. Durability / availability

- What failure domains are tolerated?
- Is durability provided by DB-level replication or underlying storage?
- How many complete database copies exist?
- Are availability copies also durability copies?
- What happens if a page server/LPS fails?

## E. Economics

- Which resources scale independently?
- Is cold data stored on expensive media?
- Does increasing write throughput force capacity scale-up?
- Does increasing capacity force redo-compute scale-up?
- How much cross-AZ traffic is generated?

This taxonomy is more useful for modern cloud databases than the simple **NoSQL vs. NewSQL** split.

---

# 40. Final assessment

## Aurora

**Best description**

> A pioneering first-generation cloud-native OLTP design that removed database-page writes from the compute/storage boundary by pushing redo, page reconstruction, recovery, and durability into a smart distributed storage service.

**Key strength**

> Exceptional multi-AZ storage durability and mature self-healing smart storage.

**Key trade-off**

> Six-way WAL replication and comparatively coupled log/page/storage functions can impose cost and constrain independent scaling.

---

## Socrates / Azure SQL Hyperscale

**Best description**

> A more finely decomposed architecture that separates synchronous log durability, asynchronous log distribution, page processing, fast local caching, and cheap durable storage.

**Key strength**

> Functional and NFR-based disaggregation allows resources to be provisioned closer to actual workload needs.

**Key trade-off**

> More distributed components, progress states, backpressure, failure/recovery workflows, and service coordination.

---

## AlloyDB

**Best description**

> A later multi-layer disaggregated PostgreSQL architecture with a dedicated regional WAL service, horizontally elastic log-processing/page servers, and sharded regional block storage.

**Key strength**

> Redo compute and block capacity can scale independently while the Primary sees a simple durable-WAL abstraction.

**Key trade-off**

> Strong three-zone replicated database state still implies substantial system-wide storage/redo work.

---

# 41. One-sentence synthesis

> **Aurora moved page work out of the database compute node; Socrates separated durability, log distribution, page processing, and cheap storage into different services; AlloyDB further made the redo/page-processing tier explicitly elastic.**

Or, in architectural shorthand:

```text
Aurora:
    Compute | Smart Storage

Socrates:
    Compute | Durable Log | Log Distribution | Page Service | Cheap Store

AlloyDB:
    Compute | Regional Log | Elastic Redo/Page Compute | Regional Block Store
```

The overall lesson is not that later architectures “eliminate” Aurora’s work. They **move the same fundamental durability and page-maintenance responsibilities behind better-separated service boundaries so that latency-sensitive work, CPU-intensive work, and capacity-intensive work can be optimized and scaled independently.**

---

# References

1. Haowen Dong, Chao Zhang, Guoliang Li, Huanchen Zhang. **Cloud-Native Databases: A Survey.** IEEE TKDE 36(12), 2024. DOI: 10.1109/TKDE.2024.3397508.  
   Survey framing: compute-storage disaggregation, “log is the database,” cloud-native OLTP/OLAP architectures.

2. Alexandre Verbitski et al. **Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases.** SIGMOD 2017.  
   https://www.amazon.science/publications/amazon-aurora-design-considerations-for-high-throughput-cloud-native-relational-databases

3. Panagiotis Antonopoulos et al. **Socrates: The New SQL Server in the Cloud.** SIGMOD 2019.  
   https://www.microsoft.com/en-us/research/wp-content/uploads/2019/05/socrates.pdf

4. Michael Haubenschild, Viktor Leis. **OLTP in the Cloud: Architectures, Tradeoffs, and Cost.** The VLDB Journal 34, Article 42, 2025.  
   https://link.springer.com/article/10.1007/s00778-025-00913-z

5. Ravi Murthy, Gurmeet Goindi. **AlloyDB for PostgreSQL under the hood: Intelligent, database-aware storage.** Google Cloud, 2022.  
   https://cloud.google.com/blog/products/databases/alloydb-for-postgresql-intelligent-scalable-storage

6. Jack Hu, Eric Lee, Prashanth Purnananda, Hanuma Kodavalla. **Scaling and Hardening XLOG: The SQL Azure Hyperscale Log Service.** ICDE 2025, pp. 4211–4221. DOI: 10.1109/ICDE65448.2025.00314.

---

## Evidence labels used implicitly in this report

- **Supported** — directly stated or measured by a cited paper/vendor architecture source.
- **Modeled** — result of the 2025 analytical cost model, not necessarily a commercial product benchmark.
- **Inference** — architectural reasoning derived from published mechanisms; not claimed as measured fact.
- **Not proven** — plausible hypothesis for which the reviewed public evidence is insufficient.

