# Chapter 7: Transactions

Making data systems reliable in the face of faults

<!--
Chapter 7 is about one of the most fundamental concepts in databases: transactions.

What is a transaction

Why are they needed

The meaning of ACID

Weak Isolation levels

Serializability
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
Data systems are hostile environments.
Many things can go wrong, and will go wrong.
...

Transactions are the database's promise: "Don't worry about all this. I've got your back."
Transactions are not always needed, we can abandon them for performance or availability in some cases.
That's why it's important to kknow what safety guarantees transactions provide and their cost.
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
...
You say "BEGIN", do a bunch of operations, and say "COMMIT".
...

If anything goes wrong in between—network failure, disk error, application crash—the database promises to roll everything back.
It's all-or-nothing.

This simplifies application code. Instead of handling every possible failure mode, you just retry on error.
-->

---

# The Rise and Fall (and Rise) of Transactions

<div class="grid grid-cols-2 gap-8 mt-8"></div>

**1975:** IBM System R introduces transactions
- Became the foundation for SQL databases
- Oracle, PostgreSQL, MySQL all follow the same model

**2010:** NoSQL movement
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

Then the NoSQL movement came along. Many of them ditched transactions entirely.
There was this belief that transactions were the enemy of scalability.

After a decade of NoSQL, people realized: writing correct applications without transactions is REALLY hard.
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
The safety guarantees provided by transactions are summarized by the ACID acronym.
Atomicity, Consistency, Isolation, Durability.

The caveat, every database interprets ACID differently.

Atomicity is fairly well-defined. Consistency is actually the application's responsibility, not the database's.
Durability is well-understood—write to disk, make replicas.

But Isolation? That's where things get messy.

Systems that do not meet ACID criteria are sometimes called "BASE" (Basically Available, Soft state, Eventual consistency). But this is pretty vague, anything that isn't ACID can be called BASE.
-->

---

# Atomicity: All-or-Nothing

what happens if something crashes mid-transaction?

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

... ... ...

If you're writing 5 records and the disk fills up after 3, you don't want 3 records sitting there.
That's inconsistent state. Atomicity says: if we can't complete, we roll back EVERYTHING.

...

Maybe abortability would be a better term than atomicity.
-->

---

# Consistency: The Odd One Out

**Not so much a database property**

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
The idea of ACID consistency is that you have certain rules that must always hold true in your data. 
...
Like, all bank accounts must be balanced.
...
But consistency depends on your application logic.
...
Consinstency is actually NOT a property of the database itself, but of the application.

The database can enforce SOME constraints—unique keys, foreign keys, not-null constraints.
But general application invariants depends on the developer.

Atomicity and isolation are properties of the DATABASE that you can rely on to keep consistency.

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
Isolation is about concurrency control. All is fine is multiple clients are reading and writing different parts of the database. How do we handle it when they access the same data?

Imagine two users incrementing a counter simultaneously.
Each reads 42, adds 1, writes 43. The final value is 43, not 44. One update is LOST.

Isolation prevents concurrent operations from interfering with each other
...
The theoretical gold standard is serializability: make it look like transactions ran one after another.
No interference, no race conditions.

But here's the problem: serializability is SLOW. Really slow.
So in practice, most databases use weaker isolation levels that allow SOME concurrency issues.

And that's where things can get messy, because these weaker can lead to subtle bugs specially if they are not properly understood.
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
Durability is the promise that once a transaction is committed, it won't be lost even if the system crashes.
But what does "durable" really mean?
...
In a single node database, durability means writing to non-volatile storage like HDD or SSD.
There is no perfect durability. If you write to one disk and the machine catches fire, your data is gone.
So we replicate.
...
But what if the whole datacenter loses power?
Or what if there's a bug that crashes all replicas on the same input?
...
There's no such thing as perfect durability. All your backups could be destroyed at the same time. There is just risk reduction: disk writes, replication, backups.

-->

---

# Single-Object vs Multi-Object Transactions
Atomicity and isolation

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

Almost every database provides atomicity and isolation for SINGLE objects.
If you're writing one document, you won't see a half-written value. That's guaranteed.
These are not transcations in the usual definition. They are often called 'lightweight transactions' but a transaction is usually understood as a mechanism for grouping multiple operations in one unit of execution.

So what about operations that touch multiple objects? Sometimes the architechture cannot avoid this.

Transferring money: deduct from Alice, add to Bob. Two updates.
Email app: insert message, increment unread counter. Two operations.

For these, you need multi-object transactions. And not all databases support them.
...
Many NoSQL databases or distributed systems abandoned multi-object transactions because they're hard to implement in distributed systems. They can get in the way of performance and availability.
But that makes application code much more complex.
-->

---

# Weak Isolation Levels
When serializable isolation is too expensive

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
Whe transaction don't touch the same data, they don't interfere and can be run in paralel. Easy.
What about when they do touch the same data? Concurrency bugs are hard to find and reproduce.
Some databases provide full serializability, it's the only way to fully avoid this but comes at a high performance cost.
...
So databases often offer weaker isolation levels that allow SOME concurrency issues but prevent others.

The problem? These levels are poorly understood. Even experienced developers don't fully grasp their implications.

This has led to real-world bugs: financial data corruption, lost money, inconsistent records.
...
We'll go through the isolation levels from weakest to strongest, and understand what bugs each one prevents.
-->

---

# Read Committed Isolation

**The most basic level** — default in PostgreSQL, Oracle, SQL Server

Two guarantees:
1. **No dirty reads**: Only see **committed** data

<v-click>

2. **No dirty writes**: Only overwrite **committed** data


</v-click>

<!--
Read Committed is the minimum viable isolation level.

No dirty reads: You won't see uncommitted changes from other transactions.
If someone updates a value but hasn't committed, you still see the old value.
...
No dirty writes: You can't overwrite a value that someone else is currently modifying.

This prevents the most obvious problems, but plenty of race conditions can still occur.
Some databases support a weaker level called Read Uncommitted, it prevents dirty writes but allows dirty reads.
-->


---

# No Dirty Reads
 

```
Transaction 1: SET x = 3
Transaction 2: GET x  → returns 2 (old value)
Transaction 1: COMMIT
Transaction 2: GET x  → returns 3 (new value)
```
<v-clicks>

**Why is it useful?**
- If a transaction updates several objects, others won't see partial updates
- If a transaction aborts, others never saw its changes
  
</v-clicks>

<!--
When a transaction has written some data but has not yet committed or aborted, and another transaction is able to read that uncommited data, this is called a dirty read.

With read committed isolation, dirty reads are prevented. 
All writes are invisible to other transactions until the writing transaction commits, as per the example.
...
Seeing the database in a partially updated state is confusing and can cause other transactions to take incorrect decisisons.
...
Without read committed isolation, if a transaction aborts, other transactions can see uncommitted changes that will never become permanent.

-->

---

# No Dirty Writes

The problem: two transactions trying to modify the same data simultaneously

<v-clicks>

<div class="mt-4">

Without read committed:
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

Transaction A starts and updates listing
Transaction B starts and commits
Transaction A updates invoices and commits
```

</div>

Read committed prevents conflicts, usually delaying the second transaction until the first commits

Read committed **does not prevent race conditions**

</v-clicks>

<!--

If two concurrect transactions try to update the same object, we don't know the order of the writes, usually the latest write wins.
If  the first write was part of an uncommitted transaction, and the second write ovewrites this uncommitted value, this is called a dirty write.
...

The book example:

Alice and Bob both try to buy the same car. Their transactions run at the same time.

Bob's update to the listing wins. Alice's update to the invoice wins.
Now Bob owns the car but Alice gets the invoice!
...
Read Committed prevents this by using locks: only one transaction can modify a row at a time.
...
Dirty writes are prevented but not race conditions, as we'll see later
-->


---

# Implementing read committed
Default in PostgreSQL, Oracle, SQL Server 2012 and more

<v-click>

Dirty writes are prevented using **locks**

</v-click>

<v-click>

## How to prevent dirty reads?

**Option 1:** Use read locks too
- **Problem:** Long write transactions block ALL readers
- **Result:** Terrible performance

</v-click>
<v-click>

**Option 2:** Remember old values (used in practice)
- Database keeps **old committed value** and **new uncommitted value**
- Readers see the **old value** until transaction commits
- Then all readers switch to seeing the **new value**

</v-click>

<!--
Read committed is a popular isolation level, it is the default in many databases like PostgreSQL, Oracle, SQL Server 2012.
...

Commonly dirty writes are prevented using Locks. A transaction that modifies a row acquires a write lock on it and other write operations must wait.
...

What about reads? An option to prevent dirty reads is to use read locks too, but this causes readers to block writers and vice versa, leading to poor performance. Transactions that only read data are blocked as well
...

Most databases use a different approach: they keep multiple versions of data.
When a transaction modifies a row, the old committed value is kept around for readers. Readers continue to see the old value until the writing transaction commits, at which point they switch to seeing the new value.


-->


---

# When Read Committed is Not Enough

**Read committed prevents dirty reads and dirty writes**

But it doesn't prevent all concurrency issues

<v-click>

**Example: Alice's bank account**
- Alice has $1,000 total: $500 in account 1, $500 in account 2
- A transaction transfers $100 from account 1 to account 2
- Alice checks her balance mid-transaction

</v-click>

<v-click>

```
Transaction: Transfer $100 from account 1 to account 2

    Alice reads account_1: $500 ✓

  BEGIN:
UPDATE account_1 SET balance = 600 
UPDATE account_2 SET balance = 400 
  COMMIT;

    Alice reads account_2: $400 x

Total she sees: $500 + $400 = $900
Actual total: $1,000
```

</v-click>

<!--
Read committed prevents obvious problems like dirty reads and writes.
But there are still plenty of race conditions it doesn't catch.
...

Book example: Alice checking her bank balance during a transfer.
She has $1,000 split across two accounts.
A transaction is moving $100 from one account to another.
...

Alice reads account 1: $500.
The transaction starts an commits commits (account 2 now has $400).
Alice reads account 2: now $400.

She sees $900 total. $100 has disappeared out of nowhere!.

This is called read skew or nonrepeatable read. If Alice reads account 1 again, she'd see $400 instead of the original $500. Under read committed, this is considered acceptable. The values Alice saw were committed at the time she read them.
Sometimes read skew is not acceptable, like during backups, analytic queries or integrity checks.
To prevent this we need snapshot isolation.
-->


---

# Snapshot Isolation and Repeatable Read

**Each transaction reads from a consistent snapshot**

<v-clicks>

The transaction sees all data that was **committed at transaction start**

Even if data changes during the transaction, you **still see the old values**

</v-clicks>

<v-click>

**Key benefits:**
- Long-running queries see **stable, consistent data**
- Much easier to **reason about query results**

</v-click>

<v-click>

**Supported by:** PostgreSQL, MySQL InnoDB, Oracle, SQL Server

</v-click>

<!--
Snapshot isolation solves read skew.

The idea is simple:
...
Each transaction reads from a consistent snapshot of the database.
You see everything that was committed when your transaction started.
...
Even if other transactions commit changes while you're running, you don't see them.
You're reading from a frozen snapshot.

This fixes the issues with read committed for:
- Long running read queries, like analytic queries
- Database backups
- Integrity checks

...
This is widely supported in modern databases.
-->


---

# Implementing Snapshot Isolation

Readers never block writers, writers never block readers

<v-clicks>

The database keeps **multiple versions** of each object

When you write, you don't overwrite—you create a **new version**

Each version is tagged with a **transaction ID**

</v-clicks>

<v-click>

**For read committed:**
- Keep 2 versions: committed value + uncommitted value

</v-click>

<v-click>

**For snapshot isolation:**
- Keep many versions, one for each transaction that needs it
- Garbage collect old versions when no transaction needs them

</v-click>

<!--
Here's how snapshot isolation works under the hood.
...

The database maintains multiple versions of each object.
This is called Multi-Version Concurrency Control, or MVCC.
...

When you update a row, the old value isn't overwritten.
Instead, a new version is created alongside it.
...

Each version is tagged with the transaction ID that created it.
...

Read committed can use a lightweight version of this—just 2 versions. Typically read isolation uses a separate snapshot for each query, snapshot isolation uses one snapshot for the entire transaction.
...

Snapshot isolation needs to keep potentially many versions around, one for each in-progress transaction.
Eventually, old versions that no transaction can see anymore are garbage collected.
-->

---


# Using MVCC to solve Alice's problem
**How versions are tracked**


Each row has:
- `created_by` — transaction ID that created this row
- `deleted_by` — transaction ID that deleted it (initially empty)

<v-click>

```
Transaction: Transfer $100 from account 1 to account 2

    Alice reads account_1: $500 - Transaction ID: 1

  BEGIN: - Transaction ID: 2
UPDATE account_1 SET balance = 600 
UPDATE account_2 SET balance = 400 
  COMMIT;

    Alice reads account_2: 
  created_by: 1; deleted_by: 2; balance: 500        <---- Alice sees this
  created_by : 2; deleted_by: NULL; balance: 400

Total she sees: $1,000
```
</v-click>

<v-click>
After Alice's transaction is finished, the version deleted by transaction 2 can be garbage collected


</v-click>


<!-- 

This is how MVCC is implemented in Postgres, similar to other databases.
Each row has metadata: which transaction created it, which transaction deleted it. The transaction IDs are monotonically increasing.
...

This time when Alice reads account 2, she sees the version created by transaction 1 (her snapshot) that was later deleted by transaction 2 (the transfer). She does not see the new version created by transaction 2 because it was created after her transaction
...

When the database is sure that no transaction needs the old version anymore, it can be garbage collected.

 -->
---

# Indexes and Snapshot Isolation

**How do indexes work with multiple versions?**

<v-clicks>

**Approach 1:** Index points to all versions
- Index query must filter out invisible versions
- Garbage collection removes old index entries

**Approach 2:** Append-only B-trees (CouchDB, Datomic, LMDB)
- Create new copy of modified pages instead of overwriting
- Each write creates a new B-tree root = consistent snapshot
- No need to filter by transaction ID
- Requires background compaction and garbage collection

</v-clicks>

<v-click>

**Performance depends on implementation details**
- PostgreSQL: Optimizations to avoid index updates when versions fit on same page

</v-click>

<!--
How do indexes work when you have multiple versions of the same object?
...

One approach: the index points to ALL versions of an object.
When you query the index, you get all versions and filter out the ones your transaction can't see.
When garbage collection removes old versions, it also removes the corresponding index entries.
...

Another approach used by some databases: append-only B-trees, also called copy-on-write B-trees.
Instead of modifying pages in place, create a new copy of each modified page.
Parent pages up to the root are copied and updated to point to the new child pages.
Unaffected pages don't need to be copied—they remain immutable.

Every write transaction creates a new B-tree root, which is a consistent snapshot at that point in time.
No need to filter based on transaction IDs because old trees remain unchanged.
However, this requires background compaction to clean up old trees.
...

The actual performance of MVCC heavily depends on implementation details.
PostgreSQL, for example, has optimizations to avoid updating indexes if different versions can fit on the same page.

As a final note.
There's a naming confusion here: many databases implement snapshot isolation but call it by different names.
Oracle calls it "serializable" (which it isn't!). PostgreSQL and MySQL call it "repeatable read."
The SQL standard doesn't have the concept of snapshot isolation—it was invented after the standard.
So databases map snapshot isolation to "repeatable read" to claim standards compliance.
The problem? The SQL standard's definition is flawed and ambiguous. Different databases provide different guarantees under "repeatable read."
IBM DB2 even uses "repeatable read" to mean serializability!
As a result, nobody really knows what "repeatable read" means. It's a mess of naming confusion.

Next up, about preventing lost updates.
-->
