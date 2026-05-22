# Chapter 12: The Future of Data Systems

<div class="text-sm text-gray-400 mt-2">
Kleppmann's vision: how things <em>should</em> be, not just how they are
</div>

<!--

- This is the author's most opinionated chapter. He switches to first person.
- The goal is not to teach new mechanics, but to synthesize everything and propose a direction.
- This chapter is different. Kleppmann stops describing how things are and starts arguing for how they should be
- Unlike every other chapter, this one is personal opinion — Kleppmann literally says 'you are welcome to disagree.'
-->

---

# What is in this chapter?

<v-clicks>

- Three big ideas:
  1. **Data Integration** — no single tool rules them all
  2. **Unbundling Databases** — compose small, specialized pieces
  3. **Doing the Right Thing** — ethics and responsibility

</v-clicks>

<!--

- [click] There are three main opinion points that cover the what, how and why of data systems design:
  - Data Integration is the "what" — combining tools.
  - Unbundling Databases is the "how" — the architecture.
  - Doing the Right Thing is the "why" — the responsibility we carry.
-->

---

# Data Integration: the real-world problem

<v-clicks>

- You will **never** have one database that does everything well
- OLTP + search index + cache + analytics + ML models = reality
- The hard question: **how do you keep all these in sync?**

</v-clicks>

<v-click>

<div class="bg-yellow-900/30 p-3 rounded mt-4 text-sm">

**Kleppmann's answer:** funnel all writes through a single ordered log (CDC / event sourcing), then derive everything else from it. Derived data > distributed transactions.

</div>
</v-click>

<!--
This is the "total order" idea from Ch 9 applied practically.

- [click] Ask: "Raise your hand if your system uses only one database for everything." 
- [click] Real life examples: R&I OLTP DB, CDC for replication, OLAP for analytics
- [click] The sync problem is where most real-world bugs live — dual writes, race conditions, stale caches. 
   - Ask: "In your team, do you have any derived datasets like indexes, caches, search indexes, ML models? How do you keep them in sync with the source of truth?"
- [click] Kleppmann's answer: a single ordered log (CDC or event sourcing) as the source of truth. Everything else is derived.
- This connects back to Ch 5 (replication logs), Ch 9 (total order broadcast), and Ch 11 (CDC).
- Ask: "Does anyone already use CDC or event sourcing? How's it going?"
-->

---

#  Distributed transactions vs. Derived data 

| | Distributed transactions | Log-based derived data |
|---|---|---|
| Ordering | Locks (2PL) | Event log |
| Atomicity | Atomic commit (2PC) | Deterministic retry + idempotence |
| Consistency | Linearizable reads | Eventual (async) |
| Fault tolerance | Abort on any failure | Contained local faults |
| Performance | Expensive coordination | Loose coupling |

<v-click>
<div class="bg-yellow-900/30 p-3 rounded mt-4 text-sm">


**Kleppmann's bet:** log-based derived data wins for most real systems.

</div>
</v-click>

<!--
This is the central technical argument of the chapter.

- Going through each row:
  - Ordering: "2PL uses locks to force a serial order. Event logs just append in order — much cheaper."
  - Atomicity: "2PC needs all participants to agree. Log-based systems retry deterministically and use idempotence."
  - Consistency: "This is the trade-off — you lose linearizable reads. But Kleppmann argues most apps don't need them."
  - Fault tolerance: "2PC aborts the whole transaction if any node fails. Log-based systems isolate failures."
  - Performance: "Coordination is expensive. Loose coupling scales better."
- [click] Kleppmann thinks log-based derived data is the future for most systems over distributed transactions. 
  - Ask: Do you agree? Have you seen this work (or fail) in practice?"
- Ask: "Has anyone been burned by 2PC or distributed transactions in practice?"
-
-->

---

# Unbundling Databases

<v-clicks>

- A database bundles many features: indexes, materialized views, replication, full-text search
- **Unbundling** = building those features as separate, composable systems
- Kleppmann's dream: `mysql | elasticsearch` — like Unix pipes for data systems

</v-clicks>

<v-click>

<div class="bg-blue-900/30 p-3 rounded mt-4 text-sm">

**The "database inside-out":** batch and stream processors are like triggers and stored procedures — but running outside the database, across an entire organization.

</div>
</v-click>

<!--
This is where the Unix philosophy (Ch 10) meets database internals (Ch 3).

- [click] Think about everything a database does internally: it builds indexes, maintains materialized views, replicates data, handles full-text search.
- [click] Unbundling means: what if those weren't features of ONE product, but separate systems composed together?
- [click] The dream syntax `mysql | elasticsearch` is Kleppmann's way of saying: CDC should be as easy as a Unix pipe. We're not there yet, but that's the direction.
- [click] The key metaphor: "CREATE INDEX inside a database is the same operation as building a search index from a Kafka topic. Both are derivation functions."
- This is what he calls "the database inside-out" — a talk he gave in 2014.
- Unbundling gives flexibility but adds operational complexity. Kleppmann acknowledges this — if one tool does everything you need, just use it.
- Ask: "Have you tried unbundling in your systems? Maybe using Kafka + ksqlDB for materialized views instead of database triggers? How did it go?"
-->

---

# The write path and the read path

```text
Write path (eager):  event -> log -> derived views (indexes, caches, models)
Read path (lazy):    query -> derived view -> response
```

<v-clicks>

- Where they **meet** = your derived dataset
- Shifting the boundary = the fundamental design choice
- More work on write path = faster reads (and vice versa)
- **This is the Twitter home timeline example from Chapter 1**

</v-clicks>

<!--
After 500 pages, Kleppmann brings us back to the fan-out vs. fan-in tradeoff from page 11. There will be tradeoffs you need to make in your systems, the correct approach depends on your use case and constraints.

- The write path is eager — it precomputes. The read path is lazy — it queries.
- [click] Where they meet is your derived dataset: an index, a cache, a materialized view.
- [click] This is the fundamental design lever: push more work to the write path and reads get faster. Push work to the read path and writes get simpler.
- [click] Examples: a full-text search index does a LOT of work on the write path (tokenizing, stemming, building inverted indexes) so reads can be fast. Grep does zero write-path work but reads are slow.
- [click] Remember the Twitter timeline from Chapter 1? Fan-out on write (precompute timelines) vs. fan-out on read (merge at query time). That was this exact tradeoff.
- Ask: In you team, do you optimize for the write path or the read path? Why?
-->

---

# Aiming for Correctness — think end to end

<v-clicks>

- Transactions (ACID) give correctness, but at high coordination cost
- **End-to-end argument:** low-level guarantees (TCP, DB transactions) are not enough
- You need **end-to-end** idempotency (client-generated request IDs)
- **Timeliness** (seeing up-to-date state) vs. **Integrity** (no corruption/data loss)

</v-clicks>

<v-click>

<div class="bg-green-900/30 p-3 rounded mt-4 text-sm">

**Key insight:** integrity is non-negotiable, but timeliness can often be relaxed. Violations of timeliness = "eventual consistency." Violations of integrity = "perpetual inconsistency."

</div>
</v-click>

<!--
This reframes the entire consistency debate from Ch 7 and Ch 9.

- [click] Chapters 7 and 9 agonized over consistency. Kleppmann now offers a cleaner lens
- [click] The end-to-end argument comes from a 1984 paper by Saltzer, Reed, and Clark. The idea: TCP deduplication, database transactions — these help, but they're not enough. You need correctness at the application boundary too.
- [click] Example: a user submits a payment form, the network times out, they click submit again. TCP won't save you. The database won't save you. You need a client-generated request ID that travels end-to-end.
- [click] Consistency consists of two things:
  - Timeliness: are you seeing the latest data? (If not, just wait — it'll catch up.)
  - Integrity: is the data correct? (If not, you have a permanent problem.)
- [click] "Eventual consistency is a timeliness violation. Data corruption is an integrity violation. Very different severity."
-->

---

# The "apology workflow" — loosely interpreted constraints

<v-clicks>

- Not every constraint needs to be enforced **before** the write
- Airlines overbook flights. Banks allow overdrafts. Hotels overbook rooms.
- **Compensating transactions**: apologize and fix it later
- This avoids coordination and enables multi-datacenter, multi-leader setups

</v-clicks>

<v-click>

<div class="bg-yellow-900/30 p-3 rounded mt-4 text-sm">

Coordination-avoiding data systems can provide strong integrity without linearizability, distributed transactions, or synchronous cross-partition coordination.

</div>
</v-click>

<!--
This is the most practically useful idea in the chapter for day-to-day system design.

- [click] We're trained to enforce constraints before the write. But Kleppmann asks: do you really need to?
- [click] Real life examples of fix it later and apologize if needed:
  - Airlines deliberately overbook. They know some people won't show up. If everyone does show up, they apologize, rebook, and give a voucher.
  - Banks allow overdrafts. They charge a fee. The cost of the apology is bounded.
  - Hotels overbook rooms. If it goes wrong, they walk you to a neighboring hotel.
- [click] The pattern is called a compensating transaction — you go ahead optimistically, then fix problems after the fact.
- [click] Why does this matter architecturally? Because if you don't need to enforce the constraint synchronously, you don't need coordination. No 2PC. No linearizability. You can run multi-datacenter, multi-leader, and still maintain integrity.
- [click]
- Ask: "Do you know scenarios in ILPT where we adopt, or could adopt, the apology workflow?"" 
-->

---

# Doing the Right Thing — ethics in data systems

<v-clicks>

- Predictive analytics can become an **"algorithmic prison"**
- Bias in training data = amplified bias in output
- **Surveillance:** try replacing "data" with "surveillance" in your job title
- Privacy is not about secrecy — it's about the **freedom to choose** what to reveal
- Data is a **toxic asset**: valuable but dangerous if breached or misused

</v-clicks>

<v-click>

<div class="bg-red-900/30 p-3 rounded mt-4 text-sm">

"Data is the pollution problem of the information age, and protecting privacy is the environmental challenge." — Bruce Schneier

</div>
</v-click>

<!--
This is where Kleppmann argues that data engineers have a responsibility to think about the ethical implications of their work. This is particularly important for predictive analytics and privacy.

- [click] Kleppmann uses the phrase 'algorithmic prison' — when someone is flagged by an algorithm, they can be systematically excluded from jobs, loans, insurance, housing. No single decision is dramatic, but the cumulative effect is devastating.
- [click] Bias in, bias out. If your training data reflects historical discrimination, your model will learn and amplify it. He calls this 'money laundering for bias.'
- [click] He proposes the thought experiment: replace 'data' with 'surveillance' everywhere. 'In our surveillance-driven organization, we collect real-time surveillance streams...' Sounds different, doesn't it?
- [click] Privacy is not secrecy. It's the freedom to choose what you reveal and to whom. When companies collect data, that right transfers from the individual to the company.
- [click] He calls data a 'toxic asset' — valuable but dangerous. If you get breached, that data harms your users. And you can't un-breach it.
- [click] The author shares this Schneier quote: "Data is the pollution problem of the information age, and protecting privacy is the environmental challenge." 
- Ask: "If you would think of data as "surveillance" in your job, would you do anything differently? What about if you thought of data as "toxic waste" instead of an asset?"
- Ask: **"Data is the pollution problem of the information age."** Is this too dramatic, or does the analogy hold? What should we as engineers actually do differently?
- Ask: "Is the Industrial Revolution analogy fair? Are we in the early days of data regulation, the way the 1800s were the early days of environmental regulation?"
-->

---

# Chapter 12 in a nutshell 

<v-clicks>

- **Data Integration:** no single tool does it all — derive everything from an ordered log
- **Unbundling:** compose specialized systems via event streams, not monolithic databases
- **Correctness:** end-to-end idempotency beats distributed transactions; integrity > timeliness
- **Constraints:** enforce strictly only when the cost of apology is unacceptable
- **Ethics:** data is power — treat it with the same caution as hazardous material

</v-clicks>

<!--
Quick recap 

- This is Chapter 12 in five sentences.
- [click] Data Integration: you'll always have multiple systems. Use an ordered log as your single source of truth and derive everything else.
- [click] Unbundling: instead of one database that does everything, compose specialized tools connected by event streams.
- [click] Correctness: don't rely on database transactions alone. Test your system end-to-end. And remember: integrity is the hard requirement, timeliness is negotiable.
- [click] Constraints: not everything needs to be checked before the write. Sometimes it's cheaper and more scalable to apologize than to coordinate.
- [click] Ethics: Kleppmann ends the book by reminding us that data about people IS power, and we carry the responsibility for how it's used.

-->

---

# The book's arc: from foundations to philosophy

```text
Ch 1–4    HOW do we store and encode data?
Ch 5–9    WHAT happens when we distribute it?
Ch 10–11  HOW do we move and transform it at scale?
Ch 12     WHY does it matter, and WHAT should we build next?
```


<v-click>

<div class="mt-12 text-center">

<div class="text-6xl mb-8">🤔</div>

**Let's discuss**
</div>

</v-click>




<!--
Connecting the very first chapter to the very last.

- The book follows an arc — from storing data, to distributing it, to moving it and transforming it, to questioning why we build it the way we do.
- Chapter 1 framed the whole book around three goals: reliability, scalability, maintainability. And presented the challenge, how do we do this?
-  Chapter 12 delivers Kleppmann's answer: derived data for scalability, loose coupling for maintainability, end-to-end thinking for reliability — and ethics for everything else.

[click] Let's now discuss the book

-->

---

# Discussion

- After reading 500+ pages: what is the **single most valuable idea** you're taking away from this book?
- What's a technical decision you made in the past that you'd now approach differently after reading this book?
- Was there a concept you were confident you understood before reading DDIA, only to realize you had it wrong?
- Kleppmann wrote this in 2017. Is DDIA already outdated? With managed cloud services abstracting away most of these concerns, do engineers still need to understand this level of detail?
- What was the hardest chapter to get through?
  - Which one was your favourite?
- Did reading as a group change how you understood the material compared to reading alone?

<!--
 
-->

---

<img src="../assets/chapter12/thatsallfolks.png" alt="bye" class="center m-auto"/>