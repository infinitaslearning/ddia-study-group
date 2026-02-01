---
theme: default
title: When MongoDB Transactions Go Wrong
info: A short production incident story
---

# When MongoDB Transactions Go Wrong

### A production lesson from MongoDB Atlas

<br/>

**Goal:** Atomic product updates  
**Result:** Database meltdown

---

## The Problem We Were Solving

- A **product** consists of **many MongoDB documents**
- Publishing must appear **atomic** to readers

### Two update strategies

1. **Full replace**
   - Delete product → write it back
   - Heavy DB load
   - Risk of missing / partial reads

2. **Delta updates**
   - Smaller writes
   - Risk of drift and orphan documents

**Idea:** Use MongoDB transactions to make full replace safe

---

## Why It Worked in Development

**Development setup**
- Node.js
- Single local MongoDB container
- No sharding
- Small datasets

**Result**
- Transactions worked fine
- No visible performance issues
- Gave a false sense of safety

> Local success ≠ production safety

---

## Production Reality (MongoDB Atlas)

**Production setup**
- MongoDB Atlas
- **3-shard cluster**
- Large products
- Many documents per product
- Real concurrency

**Key difference**
- One transaction:
  - Spanned multiple shards
  - Touched thousands of documents
  - Ran for a long time

This changed everything.

---

## What MongoDB Transactions Really Do

MongoDB uses **WiredTiger**

During a transaction:
- All writes are kept **in memory**
- Old document versions are preserved
- Changes are invisible until commit
- Cache eviction is restricted

**Large transaction = large cache footprint**

Sharded transactions multiply this cost.


---

## The WiredTiger Cache Collapse

**What went wrong**
- Huge transaction
- Many large documents
- Long execution time
- Cross-shard coordination

**Result**
- WiredTiger cache filled up
- Eviction couldn’t keep up
- Memory pressure exploded
- Writes stalled
- Reads stalled

**The cluster became unresponsive**

---

## Why It Spiraled Out of Control

- Long-running transaction blocked cleanup
- Old document versions stayed in cache
- Other operations piled on
- Feedback loop:
  - More memory → less progress
  - Less progress → more memory

> The database wasn’t slow — it was stuck

---

## Why We Only Saw This in Production

**Local MongoDB hid the problem**
- No sharding
- Small data
- Short transactions

**Atlas exposed it**
- Sharded transactions are expensive
- Real data sizes matter
- WiredTiger cache has hard limits
- Concurrency amplifies everything

---

## The Fix: No Transactions

We removed transactions entirely.

### New write order

1. **Create all new documents**
2. **Update existing documents**
   - Point to new docs
   - Remove references to soon-to-be-deleted docs
3. **Delete unreferenced documents**

---

## Why This Works Better

- No massive transaction state
- Short-lived operations
- Minimal inconsistency window
- Database remains responsive

**Trade-off:**  
Slightly weaker atomicity → massively better stability

---

## Takeaways

- MongoDB transactions are **not free**
- Large bulk rewrites inside transactions are dangerous
- Sharding amplifies transaction cost
- WiredTiger cache is a hard limit
- Careful write ordering often beats “perfect” atomicity

---

## Final Thought

> We tried to make the database do less work for our application  
> and ended up making it do much more.
