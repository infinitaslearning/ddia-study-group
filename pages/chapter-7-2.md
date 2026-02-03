
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

**1.5 Or simply force atomic operations to be executred on a single tread**

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