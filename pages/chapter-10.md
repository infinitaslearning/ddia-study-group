# Chapter 10: Batch Processing

<div class="text-sm text-gray-400 mt-2">
DDIA for senior engineers: architecture choices, trade-offs, and practical patterns
</div>

<!--
Sub-chapter: MapReduce and Distributed Filesystems
Quote: "MapReduce is a bit like Unix tools... distributed across potentially thousands of machines."
Talk track: Set framing early: this chapter is less about APIs, more about system-level trade-offs in distributed batch execution.
-->

---

# 1) System Types: Different SLAs, Different Optimizations

| System | Input/Output style | Main metric | Typical example |
|---|---|---|---|
| Online service | request -> response | availability + tail latency | checkout API |
| Batch processing | large dataset -> derived dataset | throughput + cost/job | nightly recommendations |
| Stream processing | event stream -> incremental output | freshness + lag | fraud detection pipeline |

**Rule of thumb:** batch is best when full scans and recomputation are acceptable.

<!--
Quote: "batch process typically scans over large portions of an input dataset."
Talk track: Clarify boundaries: online optimizes latency, stream optimizes freshness, batch optimizes throughput and cost per full-data pass.
-->

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

<!--
Quote: "The pattern of data processing in MapReduce is very similar" to the Unix log example.
Talk track: Use this as intuition anchor before distributed mechanics; engineers already trust pipeline composition.
-->

---

# 3) Why Unix Philosophy Matters in Data Systems

- Do one thing well -> composable jobs
- Output should be reusable -> clean interfaces between teams
- Iterate fast -> throw away bad jobs safely
- Prefer tools over heroics -> repeatable ops over one-off scripts

**Strength:** high evolvability  
**Limitation:** they run only on a single machine

<!--
Quote: "Like Unix tools, MapReduce jobs separate logic from wiring."
Talk track: Emphasize reusability and team decoupling: job logic can be stable while workflow wiring evolves.
-->

---

# 4) From Unix to MapReduce + HDFS

```text
Unix:       process + local files
MapReduce:  distributed tasks + distributed files
```

- MapReduce job ~= one distributed Unix process
- HDFS (Hadoop Distributed File System) provides shared-nothing storage on commodity machines
- Data replicated/erasure-coded for fault tolerance
- Scheduler moves compute near data (locality)

**Similarity:** immutable input, append-only output mindset

<!--
Quote: "This principle is known as putting the computation near the data."
Talk track: Concept remains shared-nothing + data locality.
-->

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

<!--
Quote: "the sort step is implicit in MapReduce."
Talk track: For seniors: sorter is both the power and the tax; many downstream design choices are about reducing this tax.
-->

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

<!--
Sub-chapter: Distributed execution of MapReduce
Quote: "partitioning by reducer, sorting, and copying ... is known as the shuffle."
Talk track: Connect cost model to bottlenecks seen in production: network saturation, spill files, and skewed reducers.
-->

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

<!--
Sub-chapter: MapReduce workflows / Materialization of Intermediate State
Quote: jobs are "less like pipelines of Unix commands" and more like temp-file stages.
Talk track: This is robust and auditable, but latency-heavy; a good fit for nightly pipelines, weaker for iterative analytics.
-->

---

# 8) Reduce-Side Join (Default, Flexible)

Real example: join clickstream events with user profile table by `user_id`

- Mappers emit `(user_id, payload)` from both datasets
- Shuffle brings same keys to same reducer
- Reducer performs sort-merge style join

**Pros:** works with arbitrary input layouts  
**Cons:** heavy shuffle and sort cost

<!--
Sub-chapter: Sort-merge joins
Quote: "all the activity events and the user record with the same user ID become adjacent."
Talk track: Main advantage is correctness under messy inputs; main downside is global data movement.
-->

---

# 9) Map-Side Joins (Faster, More Assumptions)

**Broadcast hash join**
- Small table fits in memory, shipped to every mapper

**Partitioned hash join**
- Both sides partitioned by same key/hash

**Map-side merge join**
- Both sides partitioned and sorted the same way

**Trade-off:** speed up by requiring stricter data layout contracts.

<!--
Quote: "possible to make joins faster by using a so-called map-side join."
Talk track: Stress preconditions. If partitioning/sorting contracts drift, performance or correctness assumptions collapse.
-->

---

# 10) Join Strategy Cheat Sheet

| Strategy | Best when | Main risk |
|---|---|---|
| Reduce-side | unknown/dirty input layout | expensive shuffle |
| Broadcast map-side | one side is small | memory pressure |
| Partitioned map-side | both sides co-partitioned | brittle preconditions |
| Merge map-side | both sides sorted + partitioned | high prep complexity |

**Pragmatic approach:** default to reduce-side; optimize only with measured bottlenecks.

<!--
Sub-chapter: MapReduce workflows with map-side joins
Quote: "Knowing about the physical layout of datasets ... becomes important."
Talk track: Decision should be metadata-driven: table size, partition keys, and sort order, not preference.
-->

---

# 11) Grouping + Hot Key Skew

- `GROUP BY` uses same "bring same key together" pattern
- Skewed keys (celebrities, viral products) create stragglers
- One slow reducer can dominate total job time

Mitigations:
- key salting/sharding for hot keys
- 2-stage aggregation
- selective replication for skewed joins

<!--
Sub-chapter: Handling skew
Quote: disproportionately active records are "hot keys" and can lead to "significant skew."
Talk track: Show one concrete symptom: 95% reducers finish quickly, one reducer blocks the DAG tail.
-->

---

# 12) What Batch Jobs Produce in Practice

Not just reports:
- Search indexes (e.g., Lucene/Solr segments)
- Read-only key-value datasets (recommendations, related products)
- ML feature tables and scoring artifacts

Real-life example: nightly "people you may know" index build, atomically swapped in production.

<!--
Sub-chapter: The Output of Batch Workflows / Building search indexes
Quote: "The output of a batch process is often not a report, but some other kind of structure."
Talk track: Reframe batch as data-product manufacturing, not dashboard generation.
-->

---

# 13) Side-Effect-Free Output Philosophy

- Prefer: write immutable output files, then bulk-load/swap
- Avoid: writing per-record directly into serving DB during job

Why this wins:
- Faster than per-record network writes
- Safer rollback on bad code/data
- Better operational isolation from OLTP workloads

**Core idea:** human fault tolerance through reproducible recomputation.

<!--
Sub-chapter: Key-value stores as batch process output / Philosophy of batch process outputs
Quote: "A much better solution is to build a brand-new database inside the batch job."
Talk track: Promote bulk build + atomic swap pattern for safer deploy/rollback and predictable serving impact.
-->

---

# 14) Hadoop vs MPP Databases

| Dimension | Hadoop-style stack | MPP database |
|---|---|---|
| Storage model | schema-on-read, raw files | schema-on-write, curated tables |
| Processing model | arbitrary code + many engines | SQL-first integrated engine |
| Strength | flexibility and experimentation | predictable BI performance |
| Weakness | governance/quality burden shifts to consumers | less flexible for custom workloads |

<!--
Sub-chapter: Comparing Hadoop to Distributed Databases (Diversity of storage/processing)
Quote: Hadoop is "much more like a general-purpose operating system" than classic MPP.
Talk track: Balanced view: MPP excels in curated SQL analytics; Hadoop-style stacks excel in heterogeneous data + custom compute.
-->

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

<!--
Sub-chapter: Designing for frequent faults
Quote: MapReduce retries "at the granularity of an individual task."
Talk track: Tie architecture to environment assumptions (preemption, mixed-priority clusters, long-running jobs).
-->

---

# 16) Beyond MapReduce: Dataflow Engines

Spark, Flink, Tez improve execution by treating full workflow as one graph.

Compared to classic MapReduce:
- no mandatory map/reduce alternation
- avoid unnecessary global sorts
- pipeline operators earlier
- keep more intermediate state in memory/local disk

Result: often much lower latency and less I/O.

<!--
Sub-chapter: Dataflow engines
Quote: "other tools are sometimes orders of magnitude faster" and avoid mandatory work like global sorts.
Talk track: Position Spark/Flink/Tez as execution-model upgrades, not just nicer APIs.
-->

---

# 17) Iterative Workloads and Graph Processing

- MapReduce is awkward for iterative algorithms (PageRank, graph traversal)
- Re-reading/re-writing full state every iteration is costly
- Pregel/BSP model: "think like a vertex", message passing in rounds

**Pros:** natural fit for graph algorithms  
**Limitations:** partitioning and cross-machine messaging can dominate runtime

<!--
Sub-chapter: The Pregel processing model
Quote: MapReduce "only performs a single pass over the data" (awkward for iterative convergence).
Talk track: Mention practical caveat: distributed graph processing can lose to single-node if graph fits memory/disk.
-->

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

<!--
Sub-chapter: High-Level APIs and Languages / The move toward declarative query languages
Quote: declarative joins let the framework "automatically decide" suitable join algorithms.
Talk track: End with pragmatic selection: declarative where possible, arbitrary code where necessary.
-->

