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

---

# Discussion Questions

<v-clicks>

1. Have we ever considered **transaction support** when choosing a database? Or do we pick based on other factors (speed, scalability, familiarity)?

2. When do we actually **use transactions at Infinitas**? Do we have examples? Or do we mostly accept eventual consistency and lost writes?

3. What isolation level do our production databases use? **Do we know?** Have we consciously chosen it?

4. When would you choose **performance over strict serializability**? Where have we made this trade-off?

5. Do we have any examples of **concurrency bugs** we've found in our applications? How did we discover and fix them?

6. Have we encountered **write skew** in our systems? How do we typically handle scenarios where multiple transactions read the same data and update different records (like booking conflicts, claiming unique values, or enforcing multi-record constraints)?

</v-clicks>

<!--
Let's reflect on our own systems at Infinitas.

First: When we choose MongoDB vs PostgreSQL vs whatever - do we think about transactions?
Or do we focus on performance, developer experience, and scalability?
Be honest - has transaction support ever been a deciding factor?

Second: Where DO we use transactions? Look at your codebases.
Are there places where we SHOULD be using them but aren't?
Are we accepting lost updates without realizing it?

Third: Isolation levels. Most of us probably have no idea what our databases are set to.
Is it Read Committed? Snapshot Isolation? Did we choose it or just accept the default?

Fourth: Performance vs correctness. Where have we consciously said "eventual consistency is good enough"?
Was that the right call? Any regrets?

Fifth: Concurrency bugs. These are the worst. They only show up under load.
Have we found any? How? Load testing? Production incidents?
What did we learn?

Sixth: Write skew - this one's important because it's SO common but often invisible.
Think about reservation systems, inventory management, username claims.
Have we seen this pattern? Did we recognize it as write skew?
How did we handle it - database constraints, locking, or just accepted the occasional conflict?
-->

