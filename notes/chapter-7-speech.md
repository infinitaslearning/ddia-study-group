# Chapter 7: Transactions - Presentation Speech

**Total Duration: ~20 minutes**

---

## Slide 1: Title (30 seconds)

"Welcome everyone! Today we're diving into Chapter 7 of Designing Data-Intensive Applications: Transactions.

This chapter is about one of the most fundamental concepts in databases. In a world where things constantly go wrong—disks fail, networks drop packets, processes crash—transactions are our safety net. They're the mechanism that lets us write correct applications without losing our minds worrying about every possible failure scenario.

Quick note on scope: The concepts we'll cover today apply to both single-node and distributed databases. The book saves the distributed-specific challenges—like two-phase commit and consensus protocols—for Chapter 8. Today is about the core transaction concepts that apply everywhere."

---

## Slide 2: The Harsh Reality of Data Systems (1 minute)

"Let's be honest about what we're dealing with here. Data systems—whether single-node or distributed—are hostile environments.

[Click] Database software or hardware can fail right in the middle of writing data.

[Click] Applications crash halfway through a series of operations.

[Click] Networks disconnect unexpectedly, cutting off communication.

[Click] Multiple clients simultaneously overwrite each other's changes.

[Click] Clients read data that's only partially updated.

[Click] And race conditions between clients cause bugs that are incredibly difficult to reproduce.

[Click] Without transactions, handling all these edge cases becomes impossibly complex. You'd need to write defensive code for every possible failure mode. It's exhausting just thinking about it!"

---

## Slide 3: What is a Transaction? (1 minute)

"So what IS a transaction?

[Click] Simply put, it's a way to group several reads and writes into a logical unit.

[Click] Either the entire transaction succeeds—we call this a commit—

[Click] Or it fails and gets rolled back—we call this an abort.

[Click] If it fails, the application can safely retry without worrying about partial state.

[Click] Here's an example: transferring money between accounts. We deduct from Alice's account and add to Bob's account. Both operations must succeed together, or neither should happen. That's a transaction.

The beauty of transactions is that they handle partial failures for you. Your application code becomes dramatically simpler."

---

## Slide 4: The Rise and Fall (and Rise) of Transactions (1.5 minutes)

"Transactions have had quite a journey through history.

In 1975, IBM System R introduced transactions, and they became the foundation for SQL databases. Every major database—Oracle, PostgreSQL, MySQL—followed the same model for 35 years.

Then around 2010, the NoSQL movement happened. There was this popular belief that 'transactions don't scale!' Many NoSQL databases completely abandoned transactions in pursuit of higher performance and availability.

But here we are in the present day, and we've realized that transactions ARE needed. Writing correct applications without transactions is really, really hard. So new distributed databases like Google Spanner and CockroachDB DO support transactions.

The key insight is that the question isn't 'transactions versus no transactions.' It's 'which transaction guarantees do I need, and what do they cost?'"

---

## Slide 5: ACID - The Transaction Promise (1 minute)

"You've probably heard of ACID. It stands for Atomicity, Consistency, Isolation, and Durability.

Atomicity means all-or-nothing: if an error occurs, roll back everything.

Consistency means application-defined invariants are maintained—though as we'll see, this is really the application's responsibility, not the database's.

Isolation means concurrent transactions don't interfere with each other.

And Durability means once committed, data won't be lost.

[Click] But here's the dirty secret: when a vendor says 'ACID compliant,' it's mostly a marketing term! Implementations vary wildly. Different databases interpret these guarantees very differently, especially Isolation. That's what we'll focus on today."

---

## Slide 6: Atomicity - All-or-Nothing (1 minute)

"Let's clarify what Atomicity means, because it's often misunderstood.

It's NOT about concurrency—that's what Isolation handles.

Atomicity is about fault handling. What happens if something crashes mid-transaction?

[Click] Imagine your client is writing 5 records to the database.

[Click] The disk becomes full after the 3rd write.

[Click] What happens to those 3 records that were already written?

[Click] With atomicity, the transaction aborts. All writes are discarded or undone. The database returns to its previous state, and it's safe to retry the entire transaction.

This is implemented using write-ahead logs. Every change is logged before it's applied, so rollback is possible. Without this, you'd have inconsistent partial state sitting in your database."

---

## Slide 7: Consistency - The Odd One Out (45 seconds)

"Here's something they don't tell you in database textbooks: Consistency doesn't really belong in ACID!

It's not a database property—it's about application-level invariants.

[Click] For example, in an accounting system, credits and debits must balance.

[Click] Or foreign keys must reference existing rows.

[Click] YOUR application defines what 'valid' means.

[Click] The database just provides atomicity and isolation to help you maintain consistency.

[Click] The 'C' in ACID was basically thrown in to make the acronym work. Consistency is your job as an application developer!"

---

## Slide 8: Isolation - The Hard Part (1 minute)

"Now we get to Isolation—this is where things get really interesting and really tricky.

Isolation is about concurrency control. Multiple transactions running simultaneously shouldn't interfere with each other.

Look at this example: Two users incrementing a counter. User 1 reads 42, adds 1, writes 43. User 2 reads 42, adds 1, writes 43. The final value is 43, not 44! One update is lost.

[Click] The classic textbooks say transactions should be 'serializable'—they should produce the same result as if they ran one at a time, with no concurrency.

But here's the problem: full serializability is expensive. Really expensive. So in practice, most databases use WEAKER isolation levels that allow SOME concurrency issues.

And that's where things get messy, because these weaker levels are poorly understood and lead to subtle bugs that cost companies real money."

---

## Slide 9: Durability - Promises and Reality (1 minute)

"Durability sounds simple: once committed, data should survive crashes.

[Click] In a single-node database, you write to non-volatile storage and use a write-ahead log for recovery.

[Click] In a replicated database, you copy data to multiple nodes and wait for replication before acknowledging the commit.

[Click] But here's the reality: perfect durability doesn't exist!

All replicas could fail simultaneously due to a power outage or a bug. SSDs can have firmware bugs. Disk corruption can go undetected for YEARS.

There's actually been studies showing that 30-80% of SSDs develop bad blocks in the first 4 years. And disconnected SSDs can lose data within WEEKS!

So there's no such thing as perfect durability. Only layers of risk reduction: disk writes, replication, backups. And you should use all of them together."

---

## Slide 10: Single-Object vs Multi-Object Transactions (45 seconds)

"Here's an important distinction that trips people up.

Almost every database provides atomicity and isolation for single objects. If you're writing one document or updating one row, you won't see a half-written value.

But what about operations that touch multiple objects? Like transferring money—that's deducting from Alice AND adding to Bob. Two updates. Or inserting an email message AND incrementing an unread counter.

[Click] For these multi-object operations, you need actual transaction support. And not all databases support this!

Many NoSQL databases abandoned multi-object transactions because they're hard to implement in distributed systems. But that makes application code much more complex and error-prone."

---

## Slide 11: The Email Counter Example (1 minute)

"Let me make this concrete with an example.

You're building an email app. When a new email arrives, you insert it into the database AND increment the unread counter.

On the left, we see what happens without isolation. Transaction 1 inserts a new email. Transaction 2 tries to count unread messages while Transaction 1 is still in progress. The user sees the new email in their list, but the counter shows 0 unread messages! That's confusing and wrong.

On the right, we see what happens without atomicity. The email gets inserted successfully, but then the process crashes before updating the counter. Now the counter is permanently wrong—there's an email that's not counted.

Both of these are consistency violations from the application's perspective. Transactions prevent this from happening."

---

## Slide 12: Weak Isolation Levels (30 seconds)

"So here's the situation: Full serializability is slow.

The solution most databases use is to offer weaker isolation levels.

[Click] But weaker isolation means concurrency bugs can occur.

[Click] So we need to understand what we're getting. We'll examine three levels:

[Click] Read Committed—the bare minimum. Snapshot Isolation—for consistent reads. And Serializable Isolation—the gold standard.

Let's dig into each one."

---

## Slide 13: Read Committed Isolation (1 minute)

"Read Committed is the most basic isolation level. It's the default in PostgreSQL, Oracle, and SQL Server.

It makes two guarantees: No dirty reads—you only see committed data. And no dirty writes—you only overwrite committed data.

[Click] Here's what that looks like. Transaction 1 sets x to 3. Transaction 2 tries to get x and still gets 2—the old value—because Transaction 1 hasn't committed yet. After Transaction 1 commits, Transaction 2 will see 3.

This prevents the most obvious problems, but as we'll see, plenty of race conditions can still occur with Read Committed."

---

## Slide 14: The Dirty Write Problem (1 minute)

"Let me show you why preventing dirty writes matters.

Imagine two people trying to buy the same car simultaneously. Transaction A updates the listing to show Alice as the buyer, then updates the invoice to Alice. Transaction B does the same for Bob.

Without preventing dirty writes, the transactions can interleave in a bad way. Bob's update to the listing wins, so the listing shows Bob as the buyer. But Alice's update to the invoice wins, so the invoice gets sent to Alice!

Now Bob owns the car but Alice gets charged. That's a huge problem!

Read Committed prevents this by using locks: only one transaction can modify a row at a time. When Transaction A locks the row, Transaction B must wait."

---

## Slide 15-16: Implementation Details (30 seconds - move quickly)

"How do databases implement this? For dirty writes, they use row-level locks. When you update a row, the database automatically locks it until you commit.

For dirty reads, they can't use locks because that would block readers. Instead, they use a clever trick: keep two versions of each value. Readers see the old committed version while the transaction is in progress, then switch to the new version after commit.

This is called Multi-Version Concurrency Control, or MVCC. We'll see more of this soon."

---

## Slide 17: Read Committed Isn't Enough (1 minute)

"So Read Committed prevents dirty reads, but it doesn't prevent THIS scenario.

Alice is checking her bank balance. She has $500 in Account 1 and $500 in Account 2—total $1000.

A transfer is in progress: moving $100 from Account 2 to Account 1.

[Click] Here's what Alice sees: She checks Account 1 and sees $600—the transfer has already been applied. Then she checks Account 2 and sees $500—the transfer hasn't been applied yet! Her total appears to be $1100. Money appeared from nowhere!

[Click] This is called 'read skew'—and it's completely allowed under Read Committed!

[Click] Each individual read is fine—she's seeing committed data. But the combination is inconsistent. For Alice refreshing her banking app, this is just annoying. But for long-running operations like backups or analytics queries, this is completely unacceptable."

---

## Slide 18: Snapshot Isolation (45 seconds)

"The solution is Snapshot Isolation.

[Click] Each transaction reads from a consistent snapshot of the database.

[Click] The transaction sees data as it existed at transaction start time.

[Click] Even if data is modified concurrently, the transaction doesn't see those changes. Long-running queries see a frozen point in time.

[Click] This is supported by PostgreSQL, MySQL, Oracle, and SQL Server. The key principle is: readers never block writers, and writers never block readers.

This is crucial for database backups, analytics queries that run for hours, and integrity checks that need stable data to validate against."

---

## Slide 19: Snapshot Isolation - How It Works (1 minute)

"Snapshot Isolation is implemented using Multi-Version Concurrency Control, or MVCC.

Each write creates a new version of the data, tagged with a transaction ID.

Look at this example: Transaction 12 starts, then Transaction 13 starts. Transaction 13 updates Account 2's balance from 500 to 400.

The database doesn't overwrite the old value. Instead, it marks the old row as 'deleted by transaction 13' and creates a new row with 'created by transaction 13.'

Transaction 12, which started earlier, still sees the old value: 500. Transaction 13 sees the new value: 400.

Multiple versions coexist side by side, and each transaction sees the appropriate version based on when it started. Garbage collection eventually removes old versions that no transaction needs anymore."

---

## Slide 20: Snapshot Isolation - Visibility Rules (30 seconds - move quickly)

"The visibility rules determine what each transaction can see. Essentially, you see everything that was committed when your transaction started.

[Click] At transaction start, the creating transaction must have committed.

[Click] The row must not be marked deleted, or if it is...

[Click] The deleting transaction hadn't committed yet when you started.

[Click] You ignore changes by transactions still in progress, by aborted transactions, and by transactions that started after you.

This creates a consistent snapshot frozen at your start time."

---

## Slide 21: Lost Updates Problem (1 minute)

"Now let's talk about another common concurrency bug: the lost update problem.

This happens with read-modify-write cycles, which are surprisingly common!

[Click] Incrementing a counter, updating an account balance, editing a JSON document, or two users editing a wiki page.

[Click] Look at this example: User 1 reads the counter at 42, adds 1, writes 43. User 2 reads the counter at 42, adds 1, writes 43. The expected result is 44, but the actual result is 43. One update is lost!

The second write clobbers the first. This happens all the time with bank balances, like counts, and inventory levels.

And here's the kicker: neither Read Committed nor Snapshot Isolation prevent this by default!"

---

## Slide 22: Solutions to Lost Updates (45 seconds)

"There are several ways to prevent lost updates.

[Click] Best option: Use atomic operations. Most databases support atomic increment/decrement. The database handles concurrency internally.

[Click] Second option: Explicit locking. SELECT FOR UPDATE locks the rows you're reading. Other transactions must wait. Then you can safely read-modify-write.

[Click] Third option: Automatic detection. Some databases like PostgreSQL and Oracle automatically detect lost updates and abort the transaction. The application can then retry. MySQL doesn't do this, by the way!

Choose the right tool for your use case."

---

## Slide 23: Write Skew - A Subtler Problem (1 minute)

"Now we get to write skew—this is subtler than lost updates, and it catches people off guard.

Here's the scenario: A hospital requires at least one doctor on call at all times.

[Click] Alice and Bob are both on call.

[Click] Both feel sick and both try to go off call at the same time.

[Click] Both check the database: '2 doctors on call, so it's safe for me to go off call.'

[Click] Both update their own record to mark themselves as off-call.

[Click] Result: 0 doctors on call! The constraint is violated!

Both transactions read the same data—the count of on-call doctors. Both saw 2, so both proceeded. But they updated DIFFERENT objects—Alice's record versus Bob's record.

The reads and writes don't directly conflict, but together they violate a constraint. This is write skew, and it's insidious."

---

## Slide 24: More Examples of Write Skew (30 seconds)

"Once you know about write skew, you see it everywhere.

[Click] Meeting room bookings: Check for conflicts, find none, book the room. But someone else just did the same thing. Double-booked!

[Click] Claiming a username: Check if 'alice' is taken, it's not, create account. But someone else just claimed 'alice' too. Duplicate usernames!

[Click] Multiplayer games: Check if a move is valid, it is, make the move. But the board state just changed. Invalid game state!

All of these follow the same pattern: read data, make a decision, write based on that decision. But the premise changed between the read and write."

---

## Slide 25: Preventing Write Skew (30 seconds)

"Write skew is harder to prevent than lost updates.

[Click] Option 1: Explicitly lock the rows you read with SELECT FOR UPDATE. This forces serialization.

[Click] Option 2: Use database constraints if supported. But most databases can't enforce multi-object constraints like 'at least one doctor on call.'

[Click] Option 3: Use serializable isolation. This is the real solution.

Which brings us to..."

---

## Slide 26: Serializable Isolation (1 minute)

"Serializable isolation is the strongest isolation level. It guarantees that transactions behave as if they were executed serially—one at a time.

[Click] There are three main approaches to implementing this:

[Click] Actual Serial Execution—literally run one transaction at a time. This works if transactions are fast and your dataset fits in memory.

[Click] Two-Phase Locking—pessimistic locking where transactions acquire locks and hold them until commit. Lots of waiting, but it works.

[Click] Serializable Snapshot Isolation—optimistic concurrency control that detects conflicts and aborts when necessary.

[Click] Each approach has different trade-offs in performance and scalability. The key point is: serializability IS achievable, even in distributed systems. It's just a matter of choosing the right implementation for your workload."

---

## Slide 27: Key Takeaways (1 minute)

"Let's recap the key points.

[Click] Number 1: Transactions simplify application code by handling edge cases for you. You don't need to worry about every possible failure mode.

[Click] Number 2: ACID is ambiguous. When someone says 'ACID compliant,' ask about isolation levels specifically.

[Click] Number 3: Weak isolation levels are common in production, but they lead to subtle bugs that are hard to reproduce.

[Click] Read Committed prevents dirty reads and writes. It's the bare minimum.

[Click] Snapshot Isolation prevents read skew. You need this for analytics and backups.

[Click] Serializable prevents ALL concurrency issues. It's expensive but possible.

[Click] The bottom line: Choose your isolation level consciously based on your application's consistency requirements. Don't just assume the defaults are good enough."

---

## Slide 28: Discussion Questions (2-3 minutes)

"Let's open this up for discussion.

[Click] First question: What isolation level does YOUR production database use? Do you know why that level was chosen? Most people don't even know what their database defaults to. It's worth checking!

[Click] Second: Have you encountered race conditions in production? How did you debug them? These bugs are brutal because they only happen under specific timing conditions.

[Click] Third: When would you choose performance over strict serializability? Sometimes you CAN'T afford the performance hit. When is that acceptable, and when is it reckless?

[Click] Finally: How do you test for concurrency bugs? They don't appear in unit tests. You need actual concurrent load. Chaos engineering? Load tests? What's worked for you?

I'd love to hear your thoughts and experiences."

---

## Slide 29: Thank You (30 seconds)

"And that wraps up Chapter 7 on Transactions!

This is a deep and complex topic, but I hope this gives you a solid foundation for understanding how transactions work and what guarantees they provide.

The key insight to take away is that transactions exist on a spectrum from weak to strong isolation. Understanding that spectrum is crucial for building reliable, correct systems.

Are there any questions?"

---

## Total Time Breakdown:
- Introduction & Context: ~6 minutes
- Isolation Levels Deep Dive: ~8 minutes
- Concurrency Problems: ~4 minutes
- Wrap-up & Discussion: ~2 minutes
- **Total: ~20 minutes**

## Speaking Tips:
1. **Pace yourself**: Don't rush through the complex topics (isolation levels, MVCC)
2. **Use examples**: The concrete examples (bank transfer, email counter, doctors) make abstract concepts real
3. **Pause for questions**: After slide 18 (Snapshot Isolation) is a good natural break point
4. **Be interactive**: The discussion questions at the end should generate real conversation
5. **Energy**: Keep energy high—this is complex material, so enthusiasm helps
