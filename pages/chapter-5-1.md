# Part II: Distributed Data

> "A system is distributed if the message transmission delay is not negligible compared to the time between events in a single process." — Leslie Lamport

Until now, we've discussed data stored on a **single machine**.

What happens when we need **multiple machines** to store or serve data?

<!--
Part 1 covered foundations: data models, storage engines, encoding. All assumed data lives on one machine.
Now we tackle the harder problem: what happens when one machine isn't enough?
This is where distributed systems complexity begins.
-->

---

# Why Distribute Data?

<v-clicks>

- **Scalability** — Handle more reads, writes, or data volume than a single machine can manage

- **Fault Tolerance** — Keep operating even if some machines fail

- **Latency** — Place data geographically closer to users

</v-clicks>

<!--
Scalability: Your traffic grows, your data grows. One machine has limits.
Fault tolerance: Hardware fails. Disks die. Entire datacenters can go offline. Redundancy is key.
Latency: Users in Tokyo shouldn't wait for a round-trip to a server in London. Data locality matters.
-->

---

# Scaling Up vs Scaling Out

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   SCALING UP (Vertical)              SCALING OUT (Horizontal)               │
│   ─────────────────────              ───────────────────────                │
│                                                                             │
│        ┌───────┐                     ┌───┐  ┌───┐  ┌───┐  ┌───┐            │
│        │       │                     │   │  │   │  │   │  │   │            │
│        │ BIG   │                     └───┘  └───┘  └───┘  └───┘            │
│        │ BOX   │                        Shared-Nothing Architecture         │
│        │       │                                                            │
│        └───────┘                     Each node is independent               │
│   Bigger, more expensive             Commodity hardware                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Scaling up:** Cost grows faster than linearly. Bottlenecks limit gains. Limited fault tolerance.

**Scaling out:** Shared-nothing architecture. Requires distributing data across nodes.

<!--
Scaling up: Buy a bigger machine. 2x the price rarely gives 2x the capacity due to bottlenecks (memory bandwidth, disk I/O, network). Single point of failure remains.
Scaling out: Add more machines. Each node uses its own CPU, RAM, and disks independently. No shared state between machines. This is the shared-nothing model that dominates modern distributed systems.
The catch: now you need to figure out how to distribute your data and coordinate between nodes.
-->

---

# Two Ways to Distribute Data

<div class="grid grid-cols-2 gap-8 mt-8">
<div>

### Replication
Keep a **copy** of the same data on multiple nodes

- Provides **redundancy**
- Reduces latency (serve from nearest replica)
- Increases read throughput

</div>
<div>

### Partitioning
Split a large dataset into **smaller subsets** (shards)

- Each partition lives on a different node
- Enables scaling beyond single-node capacity
- Also called *sharding*

</div>
</div>

<br>

> Often used together: each partition is replicated across multiple nodes.

<!--
Replication: Same data, multiple places. If one node dies, others have the data. Classic redundancy.
Partitioning: Different data on different nodes. A 10TB database split across 10 nodes = 1TB per node. Essential for scaling writes and storage.
In practice, you combine both: partition your data AND replicate each partition. That's how systems like Cassandra, MongoDB, and CockroachDB work.
-->

---

# Chapter 5: Replication

We'll focus on **replication** — keeping a copy of the same data on multiple machines.

Key questions:
- How do changes propagate to all replicas?
- What happens when a replica fails?
- How do we handle concurrent writes?

<!--
Chapter 5 dives deep into replication. We'll cover leader-based replication, handling node failures, replication lag, and consistency guarantees.
Chapter 6 will then cover partitioning.
These two chapters form the foundation for understanding distributed databases.
-->

---
