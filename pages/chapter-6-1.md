# Chapter 6: Partitioning

Distributing data across multiple machines

<!--
Chapter 6 builds on replication (Chapter 5) by tackling the other major strategy for distributed data: partitioning.
While replication keeps copies of the same data, partitioning splits data into distinct subsets.
-->

---

# Sharding = Partitioning

Different databases use different names for the same concept:

| Database | Term |
|----------|------|
| MongoDB | Shard |
| HBase | Region |
| Bigtable | Tablet |
| Cassandra | vnode |
| Couchbase | vBucket |

<br>

> **Partition** is the most widely adopted term. We'll use it throughout.

<!--
Don't get confused by terminology. These are all the same fundamental concept.
The industry hasn't standardized on a single term, so you'll encounter all of these.
-->

---

# Why Partition?

The main goal: **Scalability**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Single Node                        Partitioned (Shared-Nothing)           │
│   ───────────                        ────────────────────────────           │
│                                                                             │
│      ┌─────────┐                     ┌───┐  ┌───┐  ┌───┐  ┌───┐             │
│      │ ALL     │                     │ A │  │ B │  │ C │  │ D │             │
│      │ DATA    │        ──────>      │   │  │   │  │   │  │   │             │
│      │ HERE    │                     └───┘  └───┘  └───┘  └───┘             │
│      └─────────┘                     Node1  Node2  Node3  Node4             │
│                                                                             │
│   Limited by one machine             Data + queries distributed             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

- Large dataset → distributed across many **disks**
- Query load → distributed across many **processors**

<!--
With 10 nodes handling a fair share each, you theoretically get 10x the storage capacity and 10x the read/write throughput.
This is the promise of horizontal scaling through partitioning.
-->

---

# Historical Context

Partitioning was born around **1980**

<v-clicks>

- Some systems designed for **transactional workloads** (OLTP)

- Some systems designed for **analytics** (OLAP)

- Different tuning, but **same fundamentals** apply to both

</v-clicks>

<!--
The core concepts we'll cover aren't new. They've been refined over 40+ years.
Whether you're building a high-throughput transaction system or a data warehouse, these principles apply.
-->

---

# Partitioning + Replication

In practice, partitioning is **combined with replication**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Node 1              Node 2              Node 3              Node 4        │
│   ┌─────────┐         ┌─────────┐         ┌─────────┐         ┌─────────┐   │
│   │ P1 (L)  │         │ P1 (F)  │         │ P2 (L)  │         │ P2 (F)  │   │
│   │ P3 (F)  │         │ P3 (L)  │         │ P4 (F)  │         │ P4 (L)  │   │
│   └─────────┘         └─────────┘         └─────────┘         └─────────┘   │
│                                                                             │
│   L = Leader    F = Follower                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

- A record belongs to **exactly one partition**
- But it's stored on **several nodes** for fault tolerance

<!--
Each node can be leader for some partitions and follower for others.
This distributes both the data AND the write load across the cluster.
If Node 1 fails, the followers of P1 on other nodes can take over.
-->

---

# The Partitioning Goal

Spread data and query load **evenly** across nodes

<v-clicks>

- If every node takes a **fair share**...
- 10 nodes → **10x** the data capacity
- 10 nodes → **10x** the read/write throughput

</v-clicks>

<v-click>

But what if partitioning is **unfair**?

</v-click>

<!--
This is the ideal scenario. In practice, achieving perfect balance is challenging.
The next slides cover what happens when things aren't balanced.
-->

---

# Skew and Hot Spots

<div class="grid grid-cols-2 gap-8 mt-8">
<div>

### Skewed Partitioning

Some partitions have **more data or queries** than others

- Makes partitioning **less effective**
- Uneven resource utilization
- Some nodes overloaded, others idle

</div>
<div>

### Hot Spot

A partition with **disproportionately high load**

- All requests hitting the same partition
- Effectively **single-node bottleneck**
- Defeats the purpose of partitioning

</div>
</div>

<!--
Even with 10 nodes, if 90% of requests go to one partition, you haven't really scaled.
Identifying and mitigating hot spots is a key challenge in distributed systems.
-->

---

# The Naive Approach: Random Assignment

Assign records to nodes **randomly**

<v-clicks>

- Distributes data quite **evenly**

- But when reading... **where is the data?**

- You don't know → must query **all nodes in parallel**

</v-clicks>

<v-click>

> Random assignment is great for writes, terrible for reads.

</v-click>

<!--
This is why we need smarter partitioning strategies.
We need a way to know WHERE a record lives without asking every node.
-->

---

# Partitioning Strategies

How do we decide which partition gets which record?

<v-clicks>

1. **Key Range Partitioning** — Partition by ranges of the key

2. **Hash Partitioning** — Partition by hash of the key

</v-clicks>

<!--
These are the two fundamental approaches. Each has trade-offs.
Let's examine them in detail.
-->

---

# Strategy 1: Key Range Partitioning

Like organizing a **bookshelf** alphabetically

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Partition 1        Partition 2        Partition 3        Partition 4      │
│   ┌──────────┐       ┌──────────┐       ┌──────────┐       ┌──────────┐     │
│   │  A - F   │       │  G - L   │       │  M - R   │       │  S - Z   │     │
│   │          │       │          │       │          │       │          │     │
│   │ Alberto  │       │ Gianluca │       │ Nicola   │       │ Snigdha  │     │
│   │ Andrei   │       │ Jerome   │       │ Riccardo │       │ Stefano  │     │
│   │ Cristiano│       │          │       │          │       │ Wouter   │     │
│   └──────────┘       └──────────┘       └──────────┘       └──────────┘     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

- Assign each partition a **contiguous range** of keys
- When reading, you **know exactly** where to look

<!--
Simple and intuitive. Works well when you need range queries.
But there's a catch...
-->

---

# Key Range: The Skewing Problem

Partition boundaries must **adapt to the data**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   If names are uniformly distributed:     If most names start with 'S':     │
│                                                                             │
│   [A-F] [G-L] [M-R] [S-Z]                [A-F] [G-L] [M-R] [S-Z]            │
│     25%   25%   25%   25%                  5%    5%   10%   80%             │
│                                                                             │
│   ✅ Balanced                             ❌ Heavily skewed                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

- Fixed boundaries don't work with non-uniform data
- Boundaries need to be **dynamic** — not always easy

<!--
Imagine partitioning sensor data by timestamp. All recent writes hit the same partition.
Or user data by name - some letters are far more common than others.
This is why hash partitioning was invented.
-->

---

# Strategy 2: Hash Partitioning

Use a **hash function** to determine partition

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│                           hash(key) % num_partitions                        │
│                                                                             │
│   "Alberto"  ──hash──>  7294812  ──mod 4──>  Partition 0                    │
│   "Daniele"  ──hash──>  3847291  ──mod 4──>  Partition 3                    │
│   "Federico" ──hash──>  9182736  ──mod 4──>  Partition 0                    │
│   "Wouter"   ──hash──>  5729183  ──mod 4──>  Partition 3                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

- A good hash function **distributes keys evenly**
- Even **similar keys** get scattered across partitions

<!--
Hash functions turn predictable patterns into uniform distributions.
This solves the skewing problem... but introduces a new trade-off.
-->

---

# Hash Partitioning: Important Warning

Not all hash functions are suitable for partitioning!

<v-clicks>

- Hash function must be **deterministic**

- Same key → **same hash** (always!)

- Some languages' built-in hash functions vary between processes

</v-clicks>

<v-click>

> Use cryptographic hashes (MD5, SHA) or consistent hashing algorithms

</v-click>

<!--
In Java, Object.hashCode() can vary between JVM instances.
In Python, hash() is randomized by default since Python 3.3.
Always use a hash function designed for distributed systems.
-->

---

# Hash Partitioning: The Trade-off

We **lose efficient range queries**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Key Range Partitioning:                                                   │
│   "Find all users A-C" → Query Partition 1 only ✅                          │
│                                                                             │
│   Hash Partitioning:                                                        │
│   "Find all users A-C" → Query ALL partitions ❌                            │
│                                                                             │
│   Adjacent keys are scattered:                                              │
│   "Alberto" → Partition 2                                                   │
│   "Alex"    → Partition 0                                                   │
│   "Andrei"  → Partition 3                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

- Keys that were **adjacent** are now **scattered**
- Sort order is **lost**

<!--
This is a fundamental trade-off. You can't have both even distribution AND efficient range queries with a simple approach.
Some databases don't even support range queries on primary keys when using hash partitioning.
-->

---

# Database Approaches

How different databases handle this trade-off:

| Database | Range Queries on Primary Key |
|----------|------------------------------|
| MongoDB (hash sharding) | Sent to **all partitions** |
| Riak | **Not supported** (supported only via secondary indexes) |
| Voldemort | **Not supported** |

<br>

> Many databases sacrifice range queries for even distribution.

<!--
If you need range queries on your primary key, hash partitioning may not be the right choice.
But there's a clever hybrid approach...
-->

---

# Hybrid Approach: Compound Keys

**Cassandra's** clever solution

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│   Compound Primary Key: (user_id, update_timestamp)                        │
│                                                                            │
│   ┌─────────────────────────────────────────────────────────────────┐      │
│   │                                                                 │      │
│   │   hash(user_id) determines partition                            │      │
│   │   update_timestamp determines sort order within partition       │      │
│   │                                                                 │      │
│   └─────────────────────────────────────────────────────────────────┘      │
│   Partition 1              Partition 2              Partition 3            │
│                                                                            │
│   ┌────────────────┐       ┌────────────────┐       ┌────────────────┐     │
│   │ user_123       │       │ user_456       │       │ user_789       │     │
│   │  └─ ts: 10:01  │       │  └─ ts: 09:30  │       │  └─ ts: 11:00  │     │
│   │  └─ ts: 10:15  │       │  └─ ts: 09:45  │       │  └─ ts: 11:30  │     │
│   │  └─ ts: 10:30  │       │  └─ ts: 10:00  │       │  └─ ts: 12:00  │     │
│   └────────────────┘       └────────────────┘       └────────────────┘     │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

<!--
First column is hashed for partition assignment.
Remaining columns are used for sorting within the partition.
Best of both worlds - if your access patterns fit this model.
-->

---

# Compound Key Example: Social Media

Primary key: `(user_id, update_timestamp)`

<v-clicks>

- **Cannot**: Search for range of user_ids
  - `user_100 to user_200` → hits all partitions

- **Can**: Get all updates for a specific user in a time range
  - `user_123, last 24 hours` → single partition, sorted!

- Different users stored on **different partitions**

- Within each user, updates **sorted by timestamp**

</v-clicks>

<!--
This pattern is perfect for timelines, activity feeds, sensor data per device, etc.
You fix the first column and range query on subsequent columns.
The key is designing your schema around your access patterns.
-->

---

# Summary: Partitioning Strategies

| Strategy | Pros | Cons |
|----------|------|------|
| **Key Range** | Efficient range queries | Risk of skew/hot spots |
| **Hash** | Even distribution | No range queries |
| **Compound** | Best of both | Limited to specific patterns |

<br>

<v-click>

> Choose based on your **access patterns**, not just data distribution.

</v-click>

<!--
There's no universally "best" partitioning strategy.
Understanding your read and write patterns is essential for making the right choice.
Next, we'll look at more advanced topics: rebalancing, secondary indexes, and request routing.
-->

---

# Skewed Workloads and Hot Spots

Even with hash partitioning, **hot spots can still occur**

<v-clicks>

- Celebrity posts on social media → millions of interactions

- Using celebrity/post ID as partition key → **all requests hit one partition**

- Almost no database can automatically fix this

- **It's up to your application**

</v-clicks>

<!--
Think of a viral tweet or a celebrity Instagram post. Millions of people interacting with the same post_id.
Even perfect hash distribution can't help when everyone wants the same key.
-->

---

# Relieving Hot Spots: Key Splitting

A simple technique: **add random digits to hot keys**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Hot key: "celebrity_123"                                                  │
│                                                                             │
│   Split into 100 keys:                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  celebrity_123_00  →  Partition 7                                   │   │
│   │  celebrity_123_01  →  Partition 2                                   │   │
│   │  celebrity_123_02  →  Partition 5                                   │   │
│   │  ...                                                                │   │
│   │  celebrity_123_99  →  Partition 1                                   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   Writes: randomly pick suffix (00-99)                                      │
│   Reads: query ALL 100 keys, combine results                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

<!--
Adding 2 random digits splits the load across ~100 partitions.
This is a manual technique - you decide which keys need splitting.
-->

---

# Key Splitting: Trade-offs

<div class="grid grid-cols-2 gap-8 mt-8">
<div>

### Pros

- Distributes hot spot load across partitions
- Simple to implement
- Can be applied selectively

</div>
<div>

### Cons

- **Reads become expensive** — must query all split keys and combine
- Must **track which keys are split**
- Splitting non-hot keys = wasted overhead

</div>
</div>

<br>

> Only split keys you **know** are hot spots. Don't over-engineer.

<!--
You need bookkeeping to know which keys are split.
If you split everything, you're just adding overhead for no benefit.
This is a targeted solution for known hot spots.
-->

---

# Partitioning and Secondary Indexes

So far: **key-value model** — partition by primary key

But what about **secondary indexes**?

<v-clicks>

- "Find all cars where `color = red`"

- Bread and butter of **relational databases**

- Also adopted by many **NoSQL** databases

- **Problem**: secondary indexes don't map neatly to partitions

</v-clicks>

<!--
With a primary key, you know exactly which partition has your data.
Secondary indexes break this clean mapping - a "red car" could be in any partition.
-->

---

# Two Approaches for Secondary Indexes

<v-clicks>

1. **Document-based partitioning** (local index)

2. **Term-based partitioning** (global index)

</v-clicks>

<!--
Each approach has different trade-offs for reads vs writes.
Let's examine both in detail.
-->

---

# Approach 1: Document-Based (Local Index)

Each partition maintains its **own secondary index**

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                            │
│   Partition 0                         Partition 1                          │
│   ┌─────────────────────────┐         ┌─────────────────────────┐          │
│   │ Documents:              │         │ Documents:              │          │
│   │   191: color=red        │         │   306: color=red        │          │
│   │   214: color=black      │         │   768: color=yellow     │          │
│   │                         │         │                         │          │
│   │ Local Index:            │         │ Local Index:            │          │
│   │   color:red    → [191]  │         │   color:red    → [306]  │          │
│   │   color:black  → [214]  │         │   color:yellow → [768]  │          │
│   └─────────────────────────┘         └─────────────────────────┘          │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

- Write: only touch the partition containing the document
- Read: **scatter/gather** — query ALL partitions, merge results

<!--
Writes are simple - just update the local index in the same partition.
But reads are expensive - you have no idea which partitions have red cars.
-->

---

# Document-Based: Scatter/Gather

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   Query: "Find all red cars"                                            │
│                                                                         │
│                           ┌─────────┐                                   │
│                           │  Client │                                   │
│                           └────┬────┘                                   │
│                    ┌───────────┼────────────┐                           │
│                    ▼           ▼            ▼                           │
│              ┌──────────┐ ┌──────────┐ ┌─────────┐                      │
│              │ Part 0   │ │ Part 1   │ │ Part 2  │                      │
│              │ red:[191]│ │ red:[306]│ │ red:[]  │                      │
│              └────┬─────┘ └────┬─────┘ └────┬────┘                      │
│                   └────────────┼────────────┘                           │
│                                ▼                                        │
│                        Merge: [191, 306]                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> Scatter/gather can be expensive, especially with many partitions

<!--
If you have 100 partitions, you're making 100 queries for a single secondary index lookup.
Tail latency becomes a problem - you're as slow as your slowest partition.
-->

---

# Document-Based: Who Uses It?

Despite the scatter/gather overhead, it's **widely used**:

| Database | Uses Document-Based Secondary Index |
|----------|-------------------------------------|
| MongoDB | Yes |
| Cassandra | Yes |
| Riak | Yes |
| Elasticsearch | Yes |
| SolrCloud | Yes |

<br>

> If you can structure queries to hit a **single partition**, scatter/gather is avoided.

<!--
These databases accept the scatter/gather trade-off because writes stay simple.
The key is designing your data model so common queries can be served from one partition.
-->

---

# Approach 2: Term-Based (Global Index)

A **global index** covering all partitions, but **itself partitioned**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   Data Partitions                     Index Partitions                      │
│   ───────────────                     ────────────────                      │
│   ┌─────────┐ ┌─────────┐            ┌─────────────────────────────────┐    │
│   │ Part 0  │ │ Part 1  │            │ Index Partition 0 (a-r)         │    │
│   │ 191:red │ │ 306:red │  ────────> │   color:red → [191, 306]        │    │
│   │ 214:blk │ │ 768:ylw │            │   color:black → [214]           │    │
│   └─────────┘ └─────────┘            ├─────────────────────────────────┤    │
│                                      │ Index Partition 1 (s-z)         │    │
│                                      │   color:yellow → [768]          │    │
│                                      └─────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

- Index partitioned by **term** (e.g., colors a-r in partition 0, s-z in partition 1)
- Read: query **only the partition** containing your term

<!--
The term determines which index partition to query.
"color:red" - 'r' falls in a-r range, so go to index partition 0.
-->

---

# Term-Based: Partitioning Options

How to partition the global index?

<div class="grid grid-cols-2 gap-8 mt-8">
<div>

### By Term (Range)

- `color:a-r` → Partition 0
- `color:s-z` → Partition 1
- Good for **range scans**
- Risk of hot spots

</div>
<div>

### By Hash of Term

- `hash(color:red)` → Partition X
- Even distribution
- **No range queries**

</div>
</div>

<!--
Same trade-off we saw with primary key partitioning!
Range partitioning is great if you need "find all colors between blue and green".
Hash partitioning spreads load but kills range queries on the index.
-->

---

# Term-Based: Trade-offs

<div class="grid grid-cols-2 gap-8 mt-8">
<div>

### Reads: More Efficient

- Query **single partition** for a term
- No scatter/gather
- Lower latency

</div>
<div>

### Writes: More Complex

- One document may update **multiple index partitions**
- Requires **distributed writes**
- Often done **asynchronously**

</div>
</div>

<br>

<v-click>

> Synchronous updates would require distributed transactions — most DBs avoid this

</v-click>

<!--
The async update means your index might be slightly stale.
If you write a red car and immediately query for red cars, you might not see it yet.
This is a fundamental trade-off of global indexes.
-->

---

# Comparing Secondary Index Approaches

| Aspect | Document-Based (Local) | Term-Based (Global) |
|--------|------------------------|---------------------|
| **Write** | Fast (single partition) | Slow (multiple partitions) |
| **Read** | Slow (scatter/gather) | Fast (single partition) |
| **Consistency** | Immediate | Often async (eventual) |
| **Complexity** | Simpler | More complex |

<br>

<v-click>

> Choose based on your **read/write ratio** and **consistency requirements**

</v-click>

<!--
Read-heavy workloads favor global indexes.
Write-heavy workloads favor local indexes.
If you need strong consistency on reads, local indexes are safer.
-->
