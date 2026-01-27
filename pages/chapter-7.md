# Chapter 7: Transactions

Making data systems reliable in the face of faults

<!--
Chapter 7 is about one of the most fundamental concepts in databases: transactions.
In a world where things constantly go wrong—disks fail, networks drop packets, processes crash—transactions are our safety net.
They're the mechanism that lets us write correct applications without losing our minds worrying about every possible failure scenario.

Note: This chapter covers transaction concepts that apply to both single-node and distributed databases.
The specific challenges of distributed transactions (like two-phase commit and consensus) are covered in Chapter 8.
-->

---

# The Harsh Reality of Data Systems

<v-clicks>

- Database software or hardware **fails** mid-write
- Applications **crash** halfway through operations
- Networks **disconnect** unexpectedly
- Multiple clients **overwrite** each other's changes
- Clients read **partially updated** data
- **Race conditions** cause surprising bugs

</v-clicks>

<v-click>

> Without transactions, handling all these edge cases becomes **impossibly complex**

</v-click>

<!--
Let's be honest: data systems are hostile environments.
Anything that can go wrong, will go wrong.

Network cables get unplugged. Disks fail. Power cuts happen. Bugs crash your code.
And multiple users are hammering your system simultaneously, creating race conditions you never thought possible.

Transactions are the database's promise: "Don't worry about all this. I've got your back."
-->

---

# What is a Transaction?

A way to **group several reads and writes** into a logical unit

<v-clicks>

- Either the **entire transaction succeeds** (commit)
- Or it **fails and gets rolled back** (abort)
- If it fails, you can **safely retry**
- No partial failures to worry about

</v-clicks>

<v-click>

```
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE user = 'alice';
  UPDATE accounts SET balance = balance + 100 WHERE user = 'bob';
COMMIT;
```

</v-click>

<!--
Conceptually, transactions are simple.
You say "BEGIN", do a bunch of operations, and say "COMMIT".

If anything goes wrong in between—network failure, disk error, application crash—the database promises to roll everything back.
It's all-or-nothing. No half-baked state.

This dramatically simplifies application code. Instead of handling every possible failure mode, you just retry on error.
-->

---

# The Rise and Fall (and Rise) of Transactions

**1975:** IBM System R introduces transactions
- Became the foundation for SQL databases
- Oracle, PostgreSQL, MySQL all follow the same model

**~2010:** NoSQL movement
- "Transactions don't scale!"
- Many NoSQL databases abandoned transactions
- Pursued higher performance and availability

**Present day:**
- We realize transactions ARE needed
- New distributed databases support them (Spanner, CockroachDB)
- The question isn't "transactions vs no transactions"
- It's "which transaction guarantees do I need?"

<!--
Transactions have had a wild ride.

For 35 years, they were considered fundamental. Every serious database had them.

Then the NoSQL movement came along. Google and Facebook were dealing with billions of users, and relational databases weren't cutting it.
There was this belief that transactions were the enemy of scalability.

But guess what? After a decade of NoSQL, people realized: writing correct applications without transactions is REALLY hard.
So now we're seeing a return to transactions—but in distributed systems.

The key insight: it's not transactions vs no transactions. It's about understanding WHICH guarantees you need and what they cost.
-->

---

# ACID: The Transaction Promise

<div class="grid grid-cols-2 gap-8 mt-8">
<div>

**A**tomicity
All-or-nothing: if error occurs, rollback everything

**C**onsistency
Application-defined invariants are maintained

</div>
<div>

**I**solation
Concurrent transactions don't interfere with each other

**D**urability
Once committed, data won't be lost

</div>
</div>

<br>

> ⚠️ Warning: "ACID compliant" is mostly a **marketing term**. Implementations vary wildly!

<!--
You've probably heard of ACID. It's been around since 1983.

But here's the dirty secret: every database interprets ACID differently.

Atomicity is fairly well-defined. Consistency is actually the application's responsibility, not the database's.
Durability is well-understood—write to disk, make replicas.

But Isolation? That's where things get messy. And that's what we'll spend most of this chapter on.

When a vendor says "ACID compliant," ask: "Which isolation level?" Because that makes all the difference.
-->

---

# Atomicity: All-or-Nothing

**Not about concurrency** (that's isolation)

It's about **fault handling**: what happens if something crashes mid-transaction?

<v-clicks>

- Client writes 5 records
- Disk becomes full after the 3rd write
- What happens?

</v-clicks>

<v-click>

### With atomicity:
- Transaction **aborts**
- All writes are **discarded/undone**
- Database returns to **previous state**
- Safe to **retry** the entire transaction

</v-click>

<!--
Atomicity is often misunderstood because the word "atomic" means different things in different contexts.

In multi-threaded programming, atomic means "indivisible operation that can't be observed half-done."
But in ACID, atomicity is about ABORTABILITY.

If you're writing 5 records and the disk fills up after 3, you don't want 3 records sitting there.
That's inconsistent state. Atomicity says: if we can't complete, we roll back EVERYTHING.

This is implemented using write-ahead logs. Every change is logged before it's applied, so rollback is possible.
-->

---

# Consistency: The Odd One Out

**Not really a database property!**

It's about **application-level invariants**

<v-clicks>

- Example: In accounting, credits and debits must balance
- Example: Foreign keys must reference existing rows
- **Your application** defines what "valid" means
- The database just enforces **atomicity** and **isolation** to help you maintain it

</v-clicks>

<v-click>

> The "C" in ACID doesn't really belong there—it's the application's job!

</v-click>

<!--
Here's something they don't tell you in database textbooks: Consistency doesn't belong in ACID.

The database can enforce SOME constraints—unique keys, foreign keys, not-null constraints.
But general application invariants? That's on you.

If you have a rule that "total inventory across all warehouses must equal sales records," the database can't enforce that.
You need to write transactions correctly to maintain it.

Atomicity and isolation are properties of the DATABASE.
Consistency is a property of your APPLICATION.

The C was basically thrown in to make the acronym work.
-->

---

# Isolation: The Hard Part

**Concurrency control**

Multiple transactions running **simultaneously** shouldn't interfere

<div class="mt-8">

```
User 1: Read counter (42) → Add 1 → Write 43
User 2: Read counter (42) → Add 1 → Write 43
```

Result: Counter is 43, not 44! **Lost update!**

</div>

<v-click>

Classic textbooks say: transactions should be **serializable**
- Same result as if they ran **one at a time**
- But full serializability is **expensive**
- Most databases use **weaker isolation levels**

</v-click>

<!--
This is where things get really interesting—and really tricky.

Imagine two users incrementing a counter simultaneously.
Each reads 42, adds 1, writes 43. The final value is 43, not 44. One update is LOST.

The theoretical gold standard is serializability: make it look like transactions ran one after another.
No interference, no race conditions.

But here's the problem: serializability is SLOW. Really slow.
So in practice, most databases use weaker isolation levels that allow SOME concurrency issues.

And that's where things get messy, because these weaker levels are poorly understood and lead to subtle bugs.
-->

---

# Durability: Promises and Reality

Once committed, data should **survive crashes**

<v-clicks>

**Single-node database:**
- Write to non-volatile storage (HDD/SSD)
- Use write-ahead log for recovery

**Replicated database:**
- Copy data to multiple nodes
- Wait for replication before acknowledging commit

</v-clicks>

<v-click>

**But perfect durability doesn't exist:**
- All replicas could fail simultaneously (power outage, bug)
- SSDs can have firmware bugs
- Disk corruption can go undetected for years

</v-click>

<!--
Durability sounds simple: write to disk, done.

But what does "durable" really mean?

If you write to one disk and the machine catches fire, your data is gone.
So we replicate. But what if the whole datacenter loses power?
Or what if there's a bug that crashes all replicas on the same input?

And here's a fun fact: SSDs can violate their own guarantees. Studies show 30-80% develop bad blocks in the first 4 years.
Disconnected SSDs can lose data within WEEKS.

There's no such thing as perfect durability. Only layers of risk reduction: disk writes, replication, backups.
And you should use all of them together.
-->

---

# Single-Object vs Multi-Object Transactions

**Single-object operations:**
- Writing one JSON document
- Updating one row
- Incrementing one counter

**Multi-object transactions:**
- Transferring money between accounts (2 updates)
- Inserting a record and updating a counter
- Updating a document and its indexes

<v-click>

> Most databases provide atomicity/isolation for **single objects**
> 
> But multi-object transactions require **actual transaction support**

</v-click>

<!--
Here's an important distinction that trips people up.

Almost every database provides atomicity and isolation for SINGLE objects.
If you're writing one document, you won't see a half-written value. That's guaranteed.

But what about operations that touch MULTIPLE objects?

Transferring money: deduct from Alice, add to Bob. Two updates.
Email app: insert message, increment unread counter. Two operations.

For these, you need multi-object transactions. And not all databases support them.

Many NoSQL databases abandoned multi-object transactions because they're hard to implement in distributed systems.
But that makes application code much more complex.
-->

---

# The Email Counter Example

<div class="grid grid-cols-2 gap-8 mt-4">
<div>

### Without Isolation:

```sql
-- Transaction 1
INSERT INTO emails 
  (id=5, unread=true, ...)

-- Transaction 2 (concurrent)
SELECT COUNT(*) 
FROM emails 
WHERE unread = true
-- Returns wrong count!
```

User sees **new email** but counter shows **0 unread**

</div>
<div>

### Without Atomicity:

```sql
BEGIN;
  INSERT INTO emails (...);
  -- CRASH!
  UPDATE mailbox 
    SET unread_count = 
      unread_count + 1;
COMMIT;
```

Counter not updated, but email exists. **Inconsistent!**

</div>
</div>

<!--
Let's make this concrete with an example.

You're building an email app. When a new email arrives, you insert it AND increment the unread counter.

WITHOUT ISOLATION: User 2 queries the counter while User 1's transaction is in-flight.
They see the new email in the list, but the counter hasn't been incremented yet. Shows 0 unread messages!

WITHOUT ATOMICITY: The insert succeeds, but then the process crashes before updating the counter.
Now the counter is permanently wrong.

Both of these are consistency violations from the application's perspective.
Transactions prevent this.
-->

---

# Weak Isolation Levels

**Problem:** Full serializability is slow

**Solution:** Most databases use **weaker isolation levels**

<v-clicks>

But weaker isolation = **concurrency bugs**

<br>

We'll examine:
1. **Read Committed** — The bare minimum
2. **Snapshot Isolation** — For consistent reads
3. **Serializable Isolation** — The gold standard

</v-clicks>

<!--
Here's the reality: full serializability has a performance cost.

So databases offer WEAKER isolation levels that allow SOME concurrency issues but prevent others.

The problem? These levels are poorly understood. Even experienced developers don't fully grasp their implications.

This has led to real-world bugs: financial data corruption, lost money, inconsistent records.

Let's walk through the isolation levels from weakest to strongest, and understand what bugs each one prevents.
-->

---

# Read Committed Isolation

**The most basic level** — default in PostgreSQL, Oracle, SQL Server

Two guarantees:
1. **No dirty reads**: Only see **committed** data
2. **No dirty writes**: Only overwrite **committed** data

<v-click>

```
Transaction 1: SET x = 3
Transaction 2: GET x  → returns 2 (old value)
Transaction 1: COMMIT
Transaction 2: GET x  → returns 3 (new value)
```

</v-click>

<!--
Read Committed is the minimum viable isolation level.

No dirty reads: You won't see uncommitted changes from other transactions.
If someone updates a value but hasn't committed, you still see the old value.

No dirty writes: You can't overwrite a value that someone else is currently modifying.

This prevents the most obvious problems, but plenty of race conditions can still occur.
-->

---

# The Dirty Write Problem

<div class="mt-4">

```
Two people buying the same car simultaneously:

Transaction A:                Transaction B:
UPDATE listings              UPDATE listings
  SET buyer = 'Alice'          SET buyer = 'Bob'
  WHERE car_id = 123;          WHERE car_id = 123;

UPDATE invoices              UPDATE invoices
  SET buyer = 'Alice'          SET buyer = 'Bob'
  WHERE car_id = 123;          WHERE car_id = 123;

COMMIT;                      COMMIT;
```

Without preventing dirty writes:
- Listing shows **Bob** as buyer
- Invoice sent to **Alice**

</div>

<!--
This is why dirty writes are bad.

Alice and Bob both try to buy the same car. Their transactions interleave.

Bob's update to the listing wins. Alice's update to the invoice wins.
Now Bob owns the car but Alice gets the invoice!

Read Committed prevents this by using locks: only one transaction can modify a row at a time.
-->

---

# Implementation: Preventing Dirty Writes

**Row-level locks**

<v-clicks>

- Transaction wants to modify a row
- Must **acquire lock** on that row first
- Hold lock until **commit or abort**
- Other transactions must **wait**

</v-clicks>

<v-click>

This is **automatic** in Read Committed mode (and stronger levels)

</v-click>

<!--
How do databases prevent dirty writes? Locks.

When you update a row, the database automatically locks it.
Other transactions trying to modify the same row must wait.

This is automatic. You don't write lock/unlock statements.
The database handles it transparently.

But notice: this only prevents WRITES from interfering with WRITES.
What about READS?
-->

---

# Implementation: Preventing Dirty Reads

**Option 1:** Use read locks too
- **Problem:** Long write transactions block ALL readers
- **Result:** Terrible performance

<v-click>

**Option 2:** Remember old values (used in practice)
- Database keeps **old committed value** and **new uncommitted value**
- Readers see the **old value** until transaction commits
- Then all readers switch to seeing the **new value**

</v-click>

<!--
You might think: just use locks for reads too!

But that's a disaster. One slow transaction writing a value would block HUNDREDS of transactions trying to read it.

So databases use a clever trick: keep two versions.
While a transaction is modifying a value, readers still see the old committed version.
Once the transaction commits, readers switch to the new version.

This is called Multi-Version Concurrency Control (MVCC). We'll see more of this soon.
-->

---

# Read Committed Isn't Enough

Consider Alice checking her bank balance:

<div class="grid grid-cols-2 gap-4 mt-4">
<div>

```
Account 1: $500
Account 2: $500
Total: $1000
```

</div>
<div>

```
Transfer $100 from Account 2 to 1
```

</div>
</div>

<v-click>

**What Alice sees:**
1. Check Account 1: **$600** (transfer already applied)
2. Check Account 2: **$500** (transfer not yet applied)
3. Total: **$1100** ??? Money appeared from nowhere!

</v-click>

<v-click>

This is called **read skew** — allowed under Read Committed!

</v-click>

<!--
Read Committed prevents dirty reads. But it doesn't prevent THIS.

Alice checks her accounts while a transfer is in progress.
She reads Account 1 AFTER the +100, and Account 2 BEFORE the -100.

Each individual read is fine—she's seeing committed data.
But the combination is inconsistent. Money appeared from thin air!

This is called read skew or non-repeatable read.

For Alice refreshing her banking app, this is annoying but not catastrophic.
But for long-running operations like backups or analytics, this is unacceptable.
-->

---

# Snapshot Isolation

**Solution:** Each transaction reads from a **consistent snapshot**

<v-clicks>

- Transaction sees data as it existed at **transaction start**
- Even if data is modified concurrently
- Long-running queries see a **frozen point in time**

</v-clicks>

<v-click>

**Supported by:** PostgreSQL, MySQL, Oracle, SQL Server

Key principle: **Readers never block writers, writers never block readers**

</v-click>

<!--
Snapshot Isolation solves read skew.

At the start of your transaction, the database creates a snapshot.
Throughout your transaction, you see the database as it existed at that moment.

Even if other transactions commit changes, you don't see them. You're reading from a frozen snapshot.

This is crucial for:
- Database backups (can't have half-old, half-new data)
- Analytics queries (need consistent view for hours-long queries)
- Integrity checks (need stable data to validate)

And it's FAST because readers never block writers or vice versa.
-->

---

# Snapshot Isolation: How It Works

**Multi-Version Concurrency Control (MVCC)**

<div class="mt-4">

Each write creates a **new version** tagged with transaction ID

```
Transaction 12 starts
Transaction 13 starts

Transaction 13: UPDATE accounts SET balance = 400 WHERE id = 2

┌─────────────────────────────────────────────────┐
│ Account 2                                       │
│ ┌──────────────┐         ┌──────────────┐       │
│ │ balance: 500 │         │ balance: 400 │       │
│ │ created: 10  │         │ created: 13  │       │
│ │ deleted: 13  │         │ deleted: -   │       │
│ └──────────────┘         └──────────────┘       │
│   Transaction 12           Transaction 13       │
│   sees this               sees this            │
└─────────────────────────────────────────────────┘
```

</div>

<!--
Here's how it works under the hood.

Every row has metadata: which transaction created it, which transaction deleted it.

When Transaction 13 updates a row, it doesn't overwrite the old value.
Instead, it marks the old value as "deleted by transaction 13" and creates a new version "created by transaction 13."

Transaction 12, which started earlier, still sees the old version.
Transaction 13 sees the new version.

Multiple versions coexist, and each transaction sees the appropriate version based on visibility rules.

Garbage collection eventually removes old versions that no transaction needs anymore.
-->

---

# Snapshot Isolation: Visibility Rules

A transaction can see a row if:

<v-clicks>

1. At transaction start, the **creating transaction had committed**
2. The row is **not marked deleted**, OR
3. If marked deleted, the **deleting transaction hadn't committed yet**

</v-clicks>

<v-click>

**Ignore:**
- Changes by transactions **still in progress**
- Changes by **aborted** transactions
- Changes by transactions that **started later**

</v-click>

<!--
These rules determine what each transaction can see.

Essentially: you see everything that was COMMITTED when your transaction STARTED.

You ignore in-progress transactions, even if they commit later.
You ignore aborted transactions.
You ignore anything that started after you.

This creates a consistent snapshot frozen at your start time.
-->

---

# Lost Updates Problem

**Read-modify-write cycle** — surprisingly common!

<v-clicks>

- Incrementing a counter
- Updating account balance
- Editing a JSON document
- Two users editing a wiki page

</v-clicks>

<v-click>

```
User 1: Read counter (42) → +1 → Write 43
User 2: Read counter (42) → +1 → Write 43

Expected: 44
Actual: 43   ← One update lost!
```

</v-click>

<!--
Here's another common concurrency bug.

Both users read the counter at 42.
Both add 1 locally.
Both write 43.

The second write CLOBBERS the first. One update is lost.

This happens all the time: updating bank balances, like counts, inventory levels.

Neither Read Committed nor Snapshot Isolation prevent this by default!
-->

---

# Solutions to Lost Updates

**1. Atomic operations** (best if available)

```sql
UPDATE counters SET value = value + 1 WHERE key = 'foo';
```

<v-click>

**2. Explicit locking**

```sql
SELECT * FROM figures WHERE name = 'robot' FOR UPDATE;
-- Now we have exclusive lock, safe to modify
UPDATE figures SET position = 'c4' WHERE id = 1234;
```

</v-click>

<v-click>

**3. Automatic detection** (PostgreSQL, Oracle)
- Database detects lost update
- Aborts transaction
- Application retries

</v-click>

<!--
There are several ways to prevent lost updates.

BEST: Use atomic operations. Most databases support atomic increment/decrement.
The database handles concurrency internally.

GOOD: Explicit locking. SELECT FOR UPDATE locks the rows.
Other transactions must wait. Then you can safely read-modify-write.

NICE: Automatic detection. Some databases (PostgreSQL, Oracle) automatically detect lost updates and abort.
MySQL doesn't do this, by the way.

Choose the right tool for your use case.
-->

---

# Write Skew: A Subtler Problem

**The on-call doctor scenario:**

<v-clicks>

- Hospital requires **at least one doctor** on call
- Alice and Bob are both on call
- Both feel sick, both try to go off call
- Both check: "2 doctors on call, safe to proceed"
- Both update their own record
- Result: **0 doctors on call** ⚠️

</v-clicks>

<!--
This is write skew—subtler than lost updates.

Both transactions read the same data (count of on-call doctors).
Both see 2, so both proceed.
But they update DIFFERENT objects (Alice's record vs Bob's record).

The reads and writes don't directly conflict, but together they violate a constraint.

Read Committed doesn't prevent this. Snapshot Isolation doesn't prevent this.
Even automatic lost update detection doesn't help because different objects are being updated.
-->

---

# More Examples of Write Skew

<v-clicks>

**Meeting room booking:**
- Check for conflicts in time slot
- If none, book the room
- Two concurrent bookings → double-booked!

**Claiming a username:**
- Check if username is taken
- If not, create account
- Two concurrent requests → duplicate usernames!

**Multiplayer game:**
- Check if move is valid
- If yes, update game state
- Two concurrent moves → invalid game state!

</v-clicks>

<!--
Once you know about write skew, you see it everywhere.

Meeting rooms: Check for overlapping bookings, find none, insert yours.
But someone else just did the same thing. Now there are two bookings.

Usernames: Check if "alice" is taken, it's not, create account.
But someone else just claimed "alice" too.

Games: Check if move is legal given current board state, it is, make the move.
But the board state just changed.

All of these are write skew: read data, make a decision, write based on that decision.
But the premise changed between read and write.
-->

---

# Preventing Write Skew

**Options are limited:**

<v-clicks>

1. **Explicitly lock the rows** you read
   ```sql
   SELECT * FROM doctors WHERE on_call = true FOR UPDATE;
   ```

2. **Use database constraints** (if supported)
   - But most constraints are single-object
   - Multi-object constraints rarely supported

3. **Use serializable isolation** ← The real solution

</v-clicks>

<!--
Write skew is harder to prevent than lost updates.

Atomic operations don't help—multiple objects involved.
Automatic detection doesn't work—different objects updated.

Your best bet is explicitly locking the rows with FOR UPDATE.
This forces serialization.

Or use database constraints if possible. But most databases can't enforce "at least one doctor on call" as a constraint.

The real answer: serializable isolation. Which brings us to...
-->

---

# Serializable Isolation

**The strongest isolation level**

Guarantees: transactions behave as if executed **serially** (one at a time)

<v-clicks>

**Three main approaches:**
1. **Actual Serial Execution** — Literally run one transaction at a time
2. **Two-Phase Locking** — Pessimistic locking
3. **Serializable Snapshot Isolation** — Optimistic concurrency control

</v-clicks>

<v-click>

Each has different trade-offs in **performance** and **scalability**

</v-click>

<!--
Serializable isolation is the gold standard. No race conditions possible.

Transactions produce the same result as if they ran one at a time.

But how do you implement this without actually running one at a time? That would be terribly slow.

There are three main approaches, and they have VERY different performance characteristics.

Actual serial execution: Use if transactions are fast and dataset fits in memory.
Two-phase locking: Pessimistic, lots of waiting, but works for any workload.
Serializable Snapshot Isolation: Optimistic, great performance IF conflict rate is low.

We won't dive into the details now, but the key point: serializability IS achievable even in distributed systems.
-->

---

# Key Takeaways

<v-clicks>

1. **Transactions simplify application code** by handling edge cases

2. **ACID is ambiguous**—ask about isolation levels specifically

3. **Weak isolation levels** are common but lead to subtle bugs

4. **Read Committed:** Prevents dirty reads/writes (bare minimum)

5. **Snapshot Isolation:** Prevents read skew (needed for analytics)

6. **Serializable:** Prevents ALL concurrency issues (expensive but possible)

7. **Choose isolation level** based on your consistency needs

</v-clicks>

<!--
Let's recap.

Transactions aren't just an academic concept. They're essential for building correct applications.

But not all transactions are equal. The isolation level matters HUGELY.

Read Committed is the bare minimum. It prevents the most obvious bugs.
Snapshot Isolation gives you consistent snapshots for long operations.
Serializable prevents ALL race conditions but costs performance.

The key is to UNDERSTAND what guarantees you're getting.
Don't just assume "ACID compliant" means you're safe.

Choose your isolation level consciously based on your application's consistency requirements.
-->

---

# Discussion Questions

<v-clicks>

1. What isolation level does your production database use? Do you know why?

2. Have you encountered race conditions in production? How did you debug them?

3. When would you choose **performance over strict serializability**?

4. How would you test for **concurrency bugs** in your application?

</v-clicks>

<!--
Let's open this up for discussion.

First: What isolation level are you actually running in production? Most people don't even know.
Check your database defaults. You might be surprised.

Second: Race conditions are brutal to debug. They only happen under specific timing.
Anyone have war stories?

Third: Sometimes you CAN'T afford serializability. The performance hit is too high.
When is that acceptable? When is it reckless?

Finally: Testing. These bugs don't appear in unit tests. They need concurrency.
How do you test for them? Chaos engineering? Load tests?
-->

---
layout: center
class: text-center
---

# Thank You!

### Questions?

<!--
And that's Chapter 7! Transactions are deep and complex, but hopefully this gives you a solid foundation.

The key insight: transactions exist on a spectrum from weak to strong.
Understanding that spectrum is crucial for building reliable systems.

Questions?
-->
