# Chapter 10: Batch Processing

<div class="text-sm text-gray-400 mt-2">
DDIA for senior engineers: architecture choices, trade-offs, and practical patterns
</div>

---

# 1) System Types: Different SLAs, Different Optimizations

| System | Input/Output style | Main metric | Typical example |
|---|---|---|---|
| Online service | request -> response | availability + tail latency | checkout API |
| Batch processing | large dataset -> derived dataset | throughput + cost/job | nightly recommendations |
| Stream processing | event stream -> incremental output | freshness + lag | fraud detection pipeline |

**Rule of thumb:** batch is best when full scans and recomputation are acceptable.

---

# 2) Unix Pipeline: The Mental Model

```bash
cat /var/log/nginx/access.log |
awk '{print $7}' |
sort |
uniq -c |
sort -r -n |
head -n 5
```

- Many small, single-purpose stages
- Explicit dataflow via stdin/stdout
- Easy to test, rerun, and swap steps
- Real-life analog: ad-hoc SRE traffic analysis in minutes

---

# 3) Why Unix Philosophy Matters in Data Systems

- Do one thing well -> composable jobs
- Output should be reusable -> clean interfaces between teams
- Iterate fast -> throw away bad jobs safely
- Prefer tools over heroics -> repeatable ops over one-off scripts

**Strength:** high evolvability  
**Limitation:** they run only on a single machine

---

# 4) From Unix to MapReduce + HDFS

```text
Unix:       process + local files
MapReduce:  distributed tasks + distributed files
```

- MapReduce job ~= one distributed Unix process
- HDFS ((Hadoop Distributed File System) provides shared-nothing storage on commodity machines
- Data replicated/erasure-coded for fault tolerance
- Scheduler moves compute near data (locality)

**Similarity:** immutable input, append-only output mindset

---

# 5) MapReduce Execution in 4 Steps

1. Read input records
2. `map(record) -> (key, value)`
3. Shuffle/sort by key (framework-managed)
4. `reduce(key, values) -> output`

```text
Input -> Map -> Shuffle/Sort -> Reduce -> Files
```

**Key point:** map/reduce code is simple because data movement is delegated to framework.

---

# 6) Distributed Execution: Strengths and Costs

**Strengths**
- Massive parallelism without manual network code
- Automatic retry on task failures
- Deterministic-ish processing when inputs are immutable

**Costs**
- Shuffle is expensive (network + disk + sort)
- Workflow latency grows with each stage
- Full materialization between jobs creates I/O overhead

---

# 7) Workflows and Materialization

- Real pipelines often chain 10-100 jobs
- Each stage writes full intermediate files
- Next stage starts only after prior stage completes

**Pros**
- Durable checkpoints
- Easy rollback/reproducibility

**Cons**
- Stragglers delay whole workflow
- Repeated parse/sort/write cycles

---

# 8) Reduce-Side Join (Default, Flexible)

Real example: join clickstream events with user profile table by `user_id`

- Mappers emit `(user_id, payload)` from both datasets
- Shuffle brings same keys to same reducer
- Reducer performs sort-merge style join

**Pros:** works with arbitrary input layouts  
**Cons:** heavy shuffle and sort cost

---

# 9) Map-Side Joins (Faster, More Assumptions)

**Broadcast hash join**
- Small table fits in memory, shipped to every mapper

**Partitioned hash join**
- Both sides partitioned by same key/hash

**Map-side merge join**
- Both sides partitioned and sorted the same way

**Trade-off:** speed up by requiring stricter data layout contracts.

---

# 10) Join Strategy Cheat Sheet

| Strategy | Best when | Main risk |
|---|---|---|
| Reduce-side | unknown/dirty input layout | expensive shuffle |
| Broadcast map-side | one side is small | memory pressure |
| Partitioned map-side | both sides co-partitioned | brittle preconditions |
| Merge map-side | both sides sorted + partitioned | high prep complexity |

**Pragmatic approach:** default to reduce-side; optimize only with measured bottlenecks.

---

# 11) Grouping + Hot Key Skew

- `GROUP BY` uses same "bring same key together" pattern
- Skewed keys (celebrities, viral products) create stragglers
- One slow reducer can dominate total job time

Mitigations:
- key salting/sharding for hot keys
- 2-stage aggregation
- selective replication for skewed joins

---

# 12) What Batch Jobs Produce in Practice

Not just reports:
- Search indexes (e.g., Lucene/Solr segments)
- Read-only key-value datasets (recommendations, related products)
- ML feature tables and scoring artifacts

Real-life example: nightly "people you may know" index build, atomically swapped in production.

---

# 13) Side-Effect-Free Output Philosophy

- Prefer: write immutable output files, then bulk-load/swap
- Avoid: writing per-record directly into serving DB during job

Why this wins:
- Faster than per-record network writes
- Safer rollback on bad code/data
- Better operational isolation from OLTP workloads

**Core idea:** human fault tolerance through reproducible recomputation.

---

# 14) Hadoop vs MPP Databases

| Dimension | Hadoop-style stack | MPP database |
|---|---|---|
| Storage model | schema-on-read, raw files | schema-on-write, curated tables |
| Processing model | arbitrary code + many engines | SQL-first integrated engine |
| Strength | flexibility and experimentation | predictable BI performance |
| Weakness | governance/quality burden shifts to consumers | less flexible for custom workloads |

---

# 15) Fault Model: MapReduce vs MPP

**MPP tendency**
- Keep more in memory
- On node failure, often retry full query

**MapReduce tendency**
- Materialize aggressively to disk
- Retry failed tasks only

**Interpretation:**
- MapReduce optimizes for long jobs + frequent interruption
- MPP optimizes for short/interactive analytics

---

# 16) Beyond MapReduce: Dataflow Engines

Spark, Flink, Tez improve execution by treating full workflow as one graph.

Compared to classic MapReduce:
- no mandatory map/reduce alternation
- avoid unnecessary global sorts
- pipeline operators earlier
- keep more intermediate state in memory/local disk

Result: often much lower latency and less I/O.

---

# 17) Iterative Workloads and Graph Processing

- MapReduce is awkward for iterative algorithms (PageRank, graph traversal)
- Re-reading/re-writing full state every iteration is costly
- Pregel/BSP model: "think like a vertex", message passing in rounds

**Pros:** natural fit for graph algorithms  
**Limitations:** partitioning and cross-machine messaging can dominate runtime

---

# 18) High-Level APIs and Practical Selection

- Modern engines expose relational operators + UDFs
- Declarative plans let optimizers pick join order/strategy
- Custom code still needed for domain logic (ML, NLP, ranking)

```text
Decision flow:
SQL-friendly + interactive BI -> MPP / Spark SQL
Large custom pipeline + flexible formats -> Spark/Flink on data lake
Very large batch with simple robustness needs -> MapReduce still viable
```

**Takeaway:** choose the execution model by failure profile, data layout maturity, and workload shape.
