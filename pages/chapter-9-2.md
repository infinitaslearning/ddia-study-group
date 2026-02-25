# Consistency and Consensus

Part 2

---

# Total Order Broadcast

<v-clicks>

- A protocol for exchanging messages between nodes
- Requires two properties:
- **Reliable delivery**: no messages are lost; if a message is delivered to one node, it is delivered to all nodes (even if a node or the network is faulty)
- **Totally ordered delivery**: messages are delivered to every node in the same order

</v-clicks>

---

# Total Order Broadcast: where it shows up

<v-clicks>

- Consensus services (e.g., ZooKeeper)
- Database replication
- Serializable transactions
- Lock services with fencing tokens (see The leader and the lock)
- Linearizable storage (e.g., ensuring only one user can claim a unique username)

</v-clicks>

---

# Distributed Transactions and Consensus

<v-clicks>

- **Consensus** is one of the most important and fundamental problems in distributed computing
- We often need multiple nodes to **agree** on something
- Common situations where consensus matters:
- Leader election
- Atomic commit (throwback: **atomicity** from Chapter 7)

</v-clicks>

---

# Atomic commit

<v-clicks>

- Goal: a transaction **either commits everywhere or aborts everywhere**
- On a single node, atomicity is usually implemented by the **storage engine**
- Once the transaction is written to the disk log (WAL), it has (practically) been committed

</v-clicks>

---

# Atomic commit in a multi-node system

<v-clicks>

- With multiple nodes, failures and network faults can split the outcome:
- Some nodes **commit** while others **abort**
- We need a protocol to ensure a single, consistent outcome
- **Two-Phase Commit (2PC)** is a classic protocol for atomic commit across nodes

</v-clicks>

---

# Two-Phase Commit (2PC)

<v-clicks>

- Two-phase commit (**2PC**) is an algorithm for achieving **atomic transaction commits across multiple nodes**
- It introduces a new component that does not appear in single-node transactions: the **coordinator**

</v-clicks>

<v-clicks>

<div class="mt-6 p-4 bg-blue-900/30 rounded-lg">

**Coordinator:** usually implemented in a library within the application process, it's a component that handles the "source of truth" for transactions and instructs the multiple nodes on when to commit them.

</div>

</v-clicks>

---

# 2PC: process

- The application is ready to commit some data
- The coordinator sends a _prepare request_ to each node, asking them if they're able to commit the given transaction

<div class="grid grid-cols-2 gap-8 mt-8">

<div class="p-4 bg-green-900/30 rounded-lg">

## ✅ All nodes reply “yes”

- The coordinator decides to commit the transaction, and it writes the decision to disk (so that if it crashes, it can recover the decision)
- The coordinator sends a _commit request_ to the nodes
- The commit actually takes place

</div>

<div class="p-4 bg-red-900/30 rounded-lg">

## ❌ At least one node replies “no”

- The coordinator decides to abort the transaction, and it writes the decision to disk (so that if it crashes, it can recover the decision)
- The coordinator sends an _abort request_ to the nodes
- The nodes abort the transaction

</div>

</div>

---

# The risks of 2PC

<div class="mt-8 p-4 bg-red-900/30 rounded-lg">

**⚠️ Blocking atomic commit protocol**

The main issue of 2PC is that the coordinator can be a single point of failure for the whole system.
That’s why it’s called a **blocking atomic commit protocol**: if the coordinator crashes after the nodes respond to a _prepare request_, the nodes can become stuck waiting for the coordinator’s decision.

</div>
