# Discussion: Batch Processing in EdTech

<div class="mt-6 text-gray-400">
Chapter 10 themes: workflows, joins, skew, fault tolerance, and engine choices
</div>

---

# Discussion
<div class="mt-12 text-center">

<div class="text-6xl mb-8">🤔</div>

**Let's discuss:**

<v-clicks>

1. In our platforms how do you utilize batch processing? Have you ever worked with MapReduce or Spark, etc., and what were the use cases?

2. If you had to enrich clickstream events with student profile data (program, cohort, accommodations), 
when would you choose a reduce-side join over a map-side join, and why?

3. Do we have any superstar courses creating hot keys? How do we detect and mitigate skew in our batch jobs?

4. In your context, what batch outputs are most valuable: reports, search indexes, 
recommendation tables, or model features, and how do you measure business value for each?

5. Do we have any recommendation use cases in place ("next lesson" or "students like you"), 
what are the risks of writing directly from batch tasks into serving databases, and when might that still be acceptable?

</v-clicks>

</div>
---

