# Database Concepts Explained

## Table of Contents
1. [Database Isolation Levels](#database-isolation-levels)
2. [Write Skew](#write-skew)
3. [Two-Phase Locking (2PL)](#two-phase-locking-2pl)

---

## Database Isolation Levels

**Isolation levels** control what happens when multiple transactions run concurrently. They're the "I" in ACID, but they exist on a spectrum from weak to strong.

### The Four Standard Levels (Weakest → Strongest)

#### 1. Read Uncommitted (Almost never used)
- Can read data from **uncommitted** transactions
- **Problem:** Dirty reads - you see changes that might get rolled back
- **Use case:** Almost none - too dangerous

```
Transaction A: Write X = 100 (not committed)
Transaction B: Read X → sees 100
Transaction A: ROLLBACK
Transaction B: Read data that never "happened"
```

#### 2. Read Committed (Most common default)
- Only read **committed** data
- Each query sees the latest committed data

**Prevents:**
- ✅ Dirty reads
- ✅ Dirty writes

**Doesn't prevent:**
- ❌ Non-repeatable reads
- ❌ Read skew
- ❌ Lost updates
- ❌ Write skew

```
Transaction A: Read X = 100
Transaction B: Write X = 200, COMMIT
Transaction A: Read X = 200  ← Different value!
```

**Used by:** PostgreSQL (default), SQL Server, Oracle

#### 3. Repeatable Read / Snapshot Isolation
- Each transaction sees a **consistent snapshot** from when it started
- Reads don't see changes from other transactions

**Prevents:**
- ✅ Dirty reads
- ✅ Dirty writes
- ✅ Non-repeatable reads
- ✅ Read skew

**Doesn't prevent:**
- ❌ Lost updates (in most databases)
- ❌ Write skew
- ❌ Phantom reads (usually)

```
Transaction A: Starts, sees snapshot at T0
Transaction B: Write X = 200, COMMIT
Transaction A: Read X → still sees 100 (from snapshot)
```

**Used by:** MySQL (default), PostgreSQL's Repeatable Read

#### 4. Serializable (Strongest)
- Transactions behave **as if executed serially** (one at a time)
- No race conditions possible

**Prevents:**
- ✅ Everything above, plus:
- ✅ Lost updates
- ✅ Write skew
- ✅ Phantom reads
- ✅ All anomalies

**Cost:** Significant performance overhead

**Implementations:**
- **Actual serial execution** (Redis, VoltDB)
- **Two-phase locking** (MySQL, SQL Server)
- **Serializable Snapshot Isolation (SSI)** (PostgreSQL)

### Quick Comparison Table

| Level | Dirty Read | Non-repeatable Read | Phantom Read | Lost Update | Write Skew |
|-------|-----------|-------------------|-------------|------------|-----------|
| Read Uncommitted | ❌ | ❌ | ❌ | ❌ | ❌ |
| Read Committed | ✅ | ❌ | ❌ | ❌ | ❌ |
| Repeatable Read | ✅ | ✅ | ⚠️ | ⚠️ | ❌ |
| Serializable | ✅ | ✅ | ✅ | ✅ | ✅ |

### Real-World Considerations

**Default doesn't mean safe:**
- Most databases default to **Read Committed**
- This allows lost updates and write skew!
- You must explicitly choose stronger isolation

**Performance trade-offs:**
- Read Committed: Fast, but buggy under concurrency
- Snapshot Isolation: Good balance for most apps
- Serializable: Safest, but 2-10x slower

**Database differences:**
- PostgreSQL's "Repeatable Read" is actually Snapshot Isolation
- MySQL's "Repeatable Read" has different behavior than PostgreSQL
- Oracle doesn't even have Read Uncommitted

### The Key Question

**"What isolation level should I use?"**

- **Read Committed:** If you don't care about consistency much (logs, caches)
- **Snapshot Isolation:** Most production applications (balances consistency & performance)
- **Serializable:** Financial systems, inventory management, anything where correctness is critical

**Always check your database's default and set it explicitly if needed!**

---

## Write Skew

**Write skew** is a concurrency anomaly where two transactions read overlapping data, make decisions based on what they read, then write to *different* objects in ways that violate a constraint.

### The Classic Example: On-Call Doctors

```
Initial state: Alice and Bob are both on-call
Constraint: At least 1 doctor must be on-call

Transaction A (Alice):           Transaction B (Bob):
1. Read: "2 doctors on-call"     1. Read: "2 doctors on-call"
2. Check: 2 - 1 = 1 ✓ (safe)     2. Check: 2 - 1 = 1 ✓ (safe)
3. Write: Alice off-call         3. Write: Bob off-call
4. COMMIT                        4. COMMIT

Final state: 0 doctors on-call ❌ Constraint violated!
```

### Why It's Different from Lost Updates

**Lost Update:**
- Both transactions update the **same object**
- Second write overwrites the first
- Example: Both increment same counter

**Write Skew:**
- Transactions update **different objects**
- Both writes succeed
- Together they violate a business rule

### More Real-World Examples

#### 1. Meeting Room Booking
```
Both transactions check: "Room 5 free at 2pm?"
Both see: Yes
Both book: Room 5 at 2pm
Result: Double-booked room ❌
```

#### 2. Username Registration
```
Both transactions check: "Is @alice taken?"
Both see: No
Both create: User with username @alice
Result: Duplicate usernames ❌
```

#### 3. Bank Account Withdrawal
```
Account has $100, overdraft limit = 0
Both transactions check: "Balance - $80 >= 0?"
Both see: Yes ($100 - $80 = $20 ✓)
Both withdraw: $80
Result: Balance = -$60 ❌
```

### Why It's Hard to Prevent

**Standard solutions don't work:**
- ❌ Atomic operations (multiple objects involved)
- ❌ Lost update detection (different rows updated)
- ❌ Row-level locks (need to lock based on *query*, not just rows)

**What DOES work:**
1. **Explicit locking** with `SELECT FOR UPDATE`
2. **Database constraints** (if supported for your use case)
3. **Serializable isolation** (the real solution)

### The Pattern

Write skew follows this pattern:
1. Read some data
2. Make a decision based on what you read
3. Write to the database (different records than you read)
4. **The premise of your decision is now invalid** (but too late!)

It's essentially a race condition where the decision-making logic gets violated by concurrent execution.

---

## Two-Phase Locking (2PL)

**Two-Phase Locking** is a pessimistic concurrency control protocol used to achieve **serializable isolation**. It's one of the oldest and most common ways to implement serializability in databases.

### The Basic Idea

**Rule:** If a transaction wants to read/write an object that another transaction has modified, it must wait until the other transaction commits or aborts.

This is implemented using locks that are held for the **entire duration** of the transaction.

### The Two Phases

#### Phase 1: Growing Phase (Acquiring Locks)
- Transaction acquires locks as needed
- **Cannot release any locks** during this phase
- Keeps acquiring locks until it has everything it needs

#### Phase 2: Shrinking Phase (Releasing Locks)  
- Transaction releases **all locks at once** when it commits/aborts
- **Cannot acquire new locks** after starting to release

```
Transaction A:
├─ Lock X        ┐
├─ Lock Y        │ Phase 1: Growing
├─ Lock Z        ┘
├─ ... do work ...
├─ COMMIT
└─ Unlock all    ← Phase 2: Shrinking (all at once)
```

### Types of Locks

**Shared lock (S)** — for reads
- Multiple transactions can hold shared locks on the same object
- Used when you just want to read

**Exclusive lock (X)** — for writes
- Only one transaction can hold an exclusive lock
- Blocks both reads and writes from other transactions

**Lock compatibility:**
```
          Current lock
          None    S      X
Request 
  S       ✓      ✓      ✗
  X       ✓      ✗      ✗
```

### Example: Preventing Write Skew

```
Transaction A (Alice):           Transaction B (Bob):
1. SELECT ... FOR UPDATE         1. SELECT ... FOR UPDATE ← BLOCKED!
   → Acquires X lock                (waits for A's lock)
2. Read: "2 doctors"
3. Write: Alice off-call
4. COMMIT                        2. Now acquires lock
   → Releases lock               3. Read: "1 doctor" ← Sees updated data!
                                 4. Check fails: 1 - 1 = 0 ✗
                                 5. ABORT or handle error
```

Transaction B is **forced to wait** until A completes, ensuring serializability.

### Problems with Two-Phase Locking

#### 1. Performance
- Transactions block each other frequently
- Reduced concurrency = reduced throughput
- Read-heavy workloads particularly affected (readers block writers, writers block readers)

#### 2. Deadlocks
```
Transaction A:                Transaction B:
Lock row 1                   Lock row 2
... wait for row 2 ...       ... wait for row 1 ...
     ↓                            ↓
     └────── DEADLOCK! ──────────┘
```

Databases must:
- Detect deadlocks (cycle detection)
- Abort one transaction (victim selection)
- Retry the aborted transaction

#### 3. Tail Latency
- One slow transaction holding locks can block many others
- Cascading delays through the system

### Variants

**Strict Two-Phase Locking (most common)**
- Holds **all locks until commit/abort**
- Prevents cascading aborts
- This is what most databases actually use

**Predicate Locks**
- Lock based on search conditions, not just specific rows
- Prevents phantom reads
- Expensive to implement

**Index-Range Locks (next-key locking)**
- Practical approximation of predicate locks
- Lock ranges in indexes
- Used by most real databases

### Which Databases Use It?

- **MySQL/InnoDB** (when using `SERIALIZABLE` isolation)
- **SQL Server** (default for `SERIALIZABLE`)
- **DB2**

### Trade-offs

**Pros:**
- ✅ Guarantees serializability
- ✅ Works for any workload
- ✅ Well-understood and battle-tested

**Cons:**
- ❌ Significant performance overhead
- ❌ Deadlocks require detection and retry logic
- ❌ Poor concurrency compared to optimistic methods

### Alternative: Serializable Snapshot Isolation (SSI)

PostgreSQL uses **SSI** instead of 2PL:
- Optimistic approach (no blocking)
- Only aborts if conflicts detected
- Much better performance when conflicts are rare
- But still provides full serializability

**Bottom line:** 2PL is a pessimistic "better safe than sorry" approach. It prevents all race conditions but at the cost of performance and dealing with deadlocks.
