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

# Reflection Time

<v-clicks>

- How would you partition a **user activity log** that needs both:
  - Queries by user (get all activity for user X)
  - Queries by time range (get all activity in last hour)

- Can you think of a scenario where **random partitioning** might actually be acceptable?

- What happens when you need to **add more nodes** to a hash-partitioned system?

</v-clicks>

<!--
These questions preview topics we'll cover in the next session.
Think about them before moving on.
-->
