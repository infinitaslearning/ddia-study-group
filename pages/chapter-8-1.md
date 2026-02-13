# The Trouble with Distributed Systems

<div class="text-center mt-20">
<div class="text-6xl mb-8">💀</div>
<p class="text-2xl text-gray-400">Everything that can go wrong, will go wrong</p>
<p class="text-lg text-gray-500 mt-4">Chapter 8: Networks, Clocks & Partial Failures</p>
</div>

<!--
Welcome to Chapter 8: The Trouble with Distributed Systems.

This is the most pessimistic chapter in the book.

Today we'll cover unreliable networks, unreliable clocks, and why partial failures make everything hard.
-->

---

# A Pessimistic Worldview

<div class="grid grid-cols-2 gap-8 mt-8">
<div>

<v-clicks>

- In distributed systems: **suspicion, pessimism, and paranoia pay off**
- You cannot trust anything:
  - Not the network
  - Not the clocks
  - Not other nodes
- **Partial failures** are the norm, not the exception

</v-clicks>

</div>
<div class="flex items-center justify-center">

<div class="text-center">
<div class="text-8xl">🔥🐕🔥</div>
<p class="text-sm text-gray-400 mt-4">"This is fine"</p>
</div>

</div>
</div>

<!--
The key mindset for this chapter: assume everything is broken.

[click] In distributed systems, suspicion, pessimism, and paranoia pay off.

[click] You cannot trust anything. Not the network. Not the clocks. Not other nodes.

[click] Partial failures are the norm. Unlike a single computer where things work or don't, distributed systems can be partially working and partially failing at the same time.
-->

---

# What We'll Cover

<div class="mt-8">

<v-clicks>

### 1. Unreliable Networks
- Why you can't trust network requests
- Timeouts and their limitations
- Congestion and queueing

### 2. Unreliable Clocks
- Time-of-day clocks vs Monotonic clocks
- Clock drift and NTP synchronization
- Why you can't rely on timestamps

</v-clicks>

</div>

<!--
We're covering the first half of Chapter 8.

[click] First: unreliable networks. Why you can't trust requests, timeouts, and where delays come from.

[click] Second: unreliable clocks. The two types of clocks, how they drift, and why timestamps are dangerous.
-->

---
layout: section
---

# Part 1: Unreliable Networks

<p class="text-gray-400">The internet is a hostile environment</p>

<!--
Let's start with networks.

The internet was designed to be resilient. Packets take different routes, nodes can fail.

But this flexibility has a cost: no guarantees about delivery or timing.
-->

---

# Networks Are Unreliable

<div class="grid grid-cols-2 gap-8 mt-4">
<div>

**When you send a request, what can happen?**

<v-clicks>

1. ✅ Request arrives, response comes back
2. ❌ Request is lost (never arrives)
3. ❌ Request waits in a queue (delayed)
4. ❌ Remote node crashed
5. ❌ Remote node is slow (temporary overload)
6. ❌ Response is lost on the way back
7. ❌ Response is delayed in a queue

</v-clicks>

</div>
<div>

<v-click>

```
┌─────────┐          ┌─────────┐
│ Client  │──────────│ Server  │
└─────────┘          └─────────┘
     │                    │
     │   Request ──?──►   │
     │                    │
     │   ◄──?── Response  │
     │                    │

  "Did it work? 🤷"
```

</v-click>

</div>
</div>

<!--
When you send a network request, many things can happen.

[click] The happy path: request arrives, response comes back.

[click] Or the request is lost and never arrives.

[click] Or it waits in a queue somewhere.

[click] Or the remote node crashed.

[click] Or the remote node is just slow.

[click] Or the response gets lost on the way back.

[click] Or the response is delayed.

[click] Here's the key insight: from the sender's perspective, you often can't tell which one happened. All you know is you didn't get a response.
-->

---

# The Unknown Unknown

<div class="mt-8">

**The sender cannot distinguish between:**

<v-clicks>

- The request was lost
- The remote node is down
- The remote node is slow
- The response was lost

</v-clicks>

</div>

<v-click>

<div class="mt-8 p-4 bg-red-900/30 rounded-lg">

**They all look the same:** No response → Timeout

The only way to know if something succeeded: **get a positive acknowledgment**

</div>

</v-click>

<!--
This is the "unknown unknown" problem.

The sender cannot distinguish between:

[click] The request was lost.

[click] The remote node is down.

[click] The remote node is just slow.

[click] The response was lost.

[click] They all look the same: silence. No response. You wait, then timeout.

The only way to know something succeeded is to get a positive acknowledgment back.
-->

---

# Real Disaster: AWS S3 Outage (2017)

<div class="mt-8 p-6 bg-gray-800 rounded-lg">

**February 28, 2017 - "The day the internet broke"**

<v-clicks>

- An engineer ran a command to take a few servers offline
- **Typo**: removed WAY more servers than intended
- S3 in us-east-1 went down for 4+ hours
- Cascading failures: sites couldn't load images, configs, assets

</v-clicks>

</div>

<v-click>

<div class="mt-6">

**Who was affected?**
- Slack, Trello, Quora, IFTTT
- Nest thermostats stopped working
- Some sites couldn't even display error pages (hosted on S3!)

</div>

</v-click>

<!--
Let me tell you about a real disaster. February 28, 2017.

[click] An Amazon engineer was debugging S3's billing system. They ran a command to remove some servers from service.

[click] Small typo. Removed way more servers than intended.

[click] S3 in us-east-1 went down. For over 4 hours.

[click] Cascading failures everywhere. Sites couldn't load images, configs, static assets.

[click] Who was affected? Slack, Trello, Quora. Nest thermostats stopped working. Some sites couldn't even show their error pages because those were hosted on S3!

One typo took down a chunk of the internet. This is the fragility of distributed systems.
-->

---

# Timeouts: Our Only Tool

<div class="mt-8">

**Problem:** How long do you wait before giving up?

<v-clicks>

- **Too short:** False positives - declare nodes dead when they're just slow
- **Too long:** Users wait forever, system appears hung

</v-clicks>

</div>

<v-click>

<div class="mt-8 grid grid-cols-2 gap-8">
<div class="p-4 bg-yellow-900/30 rounded-lg">

**Short timeout risks:**
- Cascading failures
- Unnecessary failovers
- Duplicate actions (retry storm)

</div>
<div class="p-4 bg-blue-900/30 rounded-lg">

**Long timeout risks:**
- Poor user experience
- Slow failure detection
- Resource starvation (waiting threads)

</div>
</div>

</v-click>

<!--
Timeouts are our only tool for detecting failures. How long do you wait before giving up?

[click] Too short: you get false positives. You declare nodes dead when they're just slow.

[click] Too long: users wait forever. Your system feels hung.

[click] Let me break down the specific risks. On the left: cascading failures, unnecessary failovers, retry storms. On the right: poor user experience, slow failure detection, resource starvation. There's no perfect answer.
-->

---

# Why Delays Are Unbounded

<div class="text-center mt-4 text-xl">

**Async networks provide NO guarantee on maximum delay**

</div>

<v-click>

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        THE JOURNEY OF A PACKET                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  App ──► OS Buffer ──► NIC ──► Switch Queue ──► Router ──► ... ──►     │
│                                    │                                    │
│                          (📦 packets waiting)                          │
│                                                                         │
│  ... ──► Router ──► Switch ──► NIC ──► OS Buffer ──► App               │
│                        │                    │                           │
│               (📦 more waiting)    (even more waiting!)                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

</v-click>

<v-click>

<div class="mt-4 text-sm">

**Queueing happens at EVERY step:** network switches, routers, OS buffers, hypervisors, receiving application

</div>

</v-click>

<!--
The internet is a packet-switched network. There are NO guarantees about timing.

[click] Look at the journey of a packet. It goes through your app, OS buffer, network card, switch queues, routers, more switches, back through OS buffers, finally to the receiving app.

[click] Queueing happens at EVERY step. Network switches, routers, OS buffers, hypervisors, the receiving application. All these delays are variable and unpredictable. That's why we can't pick a "correct" timeout.
-->

---

# Congestion & Queueing in Detail

<div class="grid grid-cols-2 gap-8 mt-4">
<div>

**Where packets get delayed:**

<v-clicks>

1. **Network switch** - if port is busy, packets queue (or drop!)
2. **OS kernel** - incoming data buffered before app reads it
3. **Virtualized environments** - VM might be paused while others run
4. **TCP flow control** - sender limits rate when receiver is slow
5. **Receiving application** - GC pauses, busy threads

</v-clicks>

</div>
<div>

<v-click>

**TCP congestion control**

```
Packet loss detected
       │
       ▼
    Reduce  ◄────────────┐
    send rate            │
       │                 │
       ▼                 │
   Slowly ──► Loss? ─Yes─┘
   increase      │
    rate        No
       │         │
       ▼         ▼
    Continue  Continue
```

</v-click>

</div>
</div>

<!--
Let's go deeper on where delays come from.

[click] Network switches have limited buffer space. If traffic comes in faster than it can go out, packets queue. If the buffer fills, packets are dropped.

[click] OS kernel buffers incoming data before your app reads it.

[click] In virtualized environments, your VM might be paused while the hypervisor runs other VMs. To the guest OS, time just stops.

[click] TCP flow control limits the sender when the receiver is slow.

[click] The receiving application might be doing garbage collection. Java GC pauses can be 100ms or more.

[click] And TCP has built-in congestion control. When it detects packet loss, it slows down. Good for the network, bad for your latency.
-->

---

# UDP: When Retransmission Hurts

<div class="mt-8">

**Why does video streaming use UDP instead of TCP?**

<v-clicks>

- TCP guarantees delivery by **retransmitting** lost packets
- For video: a 100ms old frame is **useless**
- Better to skip the frame than delay the stream
- UDP: "Fire and forget" - no retransmission, no ordering, no delay

</v-clicks>

</div>

<v-click>

<div class="mt-6 p-4 bg-gray-800 rounded-lg">

**Trade-off:**
- TCP: Reliable but unpredictable latency
- UDP: Predictable latency but potential data loss

Choose based on what matters more to your application.

</div>

</v-click>

<!--
Quick aside on UDP. Why does video streaming use UDP instead of TCP?

[click] TCP guarantees delivery by retransmitting lost packets.

[click] For video, a 100ms old frame is useless. The video has moved on.

[click] Better to skip the frame than delay the stream.

[click] UDP is "fire and forget". No retransmission, no ordering, no delay.

[click] It's a trade-off. TCP: reliable but unpredictable latency. UDP: predictable latency but potential data loss. VoIP, gaming, streaming all use UDP for this reason.
-->

---

# Dynamic Timeouts

<div class="mt-8">

**Static timeouts are suboptimal - conditions change!**

<v-clicks>

**Solution: Adapt based on observed behavior**

- Measure response times over time
- Track variance, not just average
- Adjust timeout dynamically

</v-clicks>

</div>

<!--
Static timeouts are suboptimal. Network conditions change!

[click] The solution: adapt based on observed behavior.

[click] Measure response times over time.

[click] Track variance, not just average.

[click] Adjust timeout dynamically.
-->

---

# Synchronous vs Asynchronous Networks

<div class="grid grid-cols-2 gap-8 mt-8">
<div>

**Phone Networks (Circuit-switched)**

<v-clicks>

- Dedicated bandwidth per call
- **Guaranteed** capacity end-to-end
- Fixed latency (bounded delay)
- Inefficient: bandwidth wasted when silent

</v-clicks>

</div>
<div>

**Internet (Packet-switched)**

<v-clicks>

- Share bandwidth dynamically
- **No guarantees** - best effort only
- Variable latency (unbounded delay)
- Efficient: only use what you need

</v-clicks>

</div>
</div>

<v-click>

<div class="mt-6 p-4 bg-blue-900/30 rounded-lg text-center">

The internet **chose** to be unreliable in exchange for efficiency

</div>

</v-click>

<!--
Why is the internet unreliable? Compare it to phone networks.

Phone networks are circuit-switched.

[click] Dedicated bandwidth per call.

[click] Guaranteed capacity end-to-end.

[click] Fixed latency - bounded delay.

[click] But inefficient: bandwidth wasted when you're silent.

[click] Now the internet - packet-switched. Share bandwidth dynamically.

[click] No guarantees - best effort only.

[click] Variable latency - unbounded delay.

[click] But efficient: only use what you need.

[click] The internet CHOSE to be unreliable in exchange for efficiency. It was a deliberate design decision.
-->

---

# Can We Make the Internet More Reliable?

<div class="mt-8">

**QoS (Quality of Service) and Admission Control**

<v-clicks>

- Prioritize certain types of traffic
- Reserve bandwidth for real-time applications
- Rate-limit senders to prevent congestion
- **Can** emulate circuit-switching behavior

</v-clicks>

</div>

<v-click>

<div class="mt-6 p-4 bg-yellow-900/30 rounded-lg">

**But in practice:**
- Internet QoS is rarely enabled
- Requires cooperation across networks
- Multi-tenant datacenters can't predict neighbor traffic
- Cost/benefit often doesn't justify the complexity

</div>

</v-click>

<!--
Can we make the internet more reliable? There are techniques.

QoS - Quality of Service - and Admission Control.

[click] Prioritize certain types of traffic.

[click] Reserve bandwidth for real-time applications.

[click] Rate-limit senders to prevent congestion.

[click] You CAN emulate circuit-switching behavior.

[click] But in practice, almost nobody does this. Requires cooperation across networks. In cloud environments, you can't control your neighbor's traffic. Cost-benefit often doesn't work out.

So we're stuck with unreliable networks. Build your systems accordingly.
-->

---
layout: section
---

# Part 2: Unreliable Clocks

<p class="text-gray-400">"Time is an illusion. Lunchtime doubly so." - Douglas Adams</p>

<!--
Now let's talk about clocks.

In a distributed system, "what time is it?" is a surprisingly hard question.

Each machine has its own clock. They're not synchronized. They drift.
-->

---

# Why Do We Need Clocks?

<div class="grid grid-cols-2 gap-8 mt-8">
<div>

**Duration measurement**

<v-clicks>

- How long did this request take?
- Has the timeout expired?
- When should the cache entry expire?

</v-clicks>

</div>
<div>

**Points in time**

<v-clicks>

- When was this record created?
- Which write happened first?
- Is this certificate still valid?

</v-clicks>

</div>
</div>

<v-click>

<div class="mt-8 p-4 bg-red-900/30 rounded-lg">

**Problem:** Computers have two types of clocks, and both have issues

</div>

</v-click>

<!--
Clocks serve two purposes in distributed systems.

[click] Duration measurement: How long did this request take?

[click] Has the timeout expired?

[click] When should cache expire?

[click] Points in time: When was this record created?

[click] Which write happened first?

[click] Is this certificate still valid?

[click] The problem: computers have TWO different kinds of clocks. They work differently. And both have issues.
-->

---

# How Computers Keep Time

<div class="mt-8">

**Quartz Crystal Oscillator**

<v-clicks>

- Every computer has one
- Vibrates at a known frequency
- Counts oscillations → tracks time
- **Problem:** They drift!

</v-clicks>

</div>

<v-click>

<div class="mt-6 p-4 bg-gray-800 rounded-lg">

**Clock Drift**

```
Typical drift: ~35ms per day (good crystal)
Worst case:   ~17 seconds per day (cheap crystal or temperature variations)

Over a month: Could be off by minutes!
```

</div>

</v-click>

<!--
Every computer has a quartz crystal oscillator.

[click] It vibrates at a known frequency.

[click] Count oscillations, you get time.

[click] Simple, but crystals aren't perfect.

[click] The problem: they drift!

[click] A good crystal might drift 35 milliseconds per day. A cheap one, or one in a hot server room, might drift 17 SECONDS per day. Leave a server running for a month? It could be minutes off.
-->

---

# NTP: Network Time Protocol

<div class="mt-4">

**How machines synchronize their clocks**

<v-clicks>

- Connect to servers with accurate time (often GPS-based)
- Measure round-trip delay, estimate offset
- Adjust local clock to match

</v-clicks>

</div>

<v-click>

```
┌──────────┐                    ┌──────────┐
│  Client  │ ── "What time?" ──►│   NTP    │
│          │                    │  Server  │
│  12:00:00│ ◄── "12:00:01" ────│ 12:00:01 │
└──────────┘                    └──────────┘
      │
      ▼
  Adjust clock by ~1 second
  (accounting for network delay)
```

</v-click>

<v-click>

<div class="mt-4 p-4 bg-yellow-900/30 rounded-lg">

**NTP problems:** Network delays vary, leap seconds cause jumps, misconfigured servers exist

</div>

</v-click>

<!--
NTP - Network Time Protocol - is how computers synchronize their clocks.

[click] Connect to servers with accurate time, often GPS-based.

[click] Measure round-trip delay, estimate offset.

[click] Adjust local clock to match.

[click] Here's how it works visually. Your computer asks "what time is it?" The NTP server responds. You adjust your clock, accounting for network delay.

[click] But there are problems. Network delays vary. Leap seconds cause jumps. Misconfigured servers exist. NTP gets you within tens of milliseconds. Not bad, but in distributed systems, that can matter.
-->

---

# Time-of-Day Clocks

<div class="mt-8">

**Returns "wall clock" time - what you'd see on a clock**

<v-clicks>

- `System.currentTimeMillis()` (Java)
- `Date.now()` (JavaScript)
- `time.time()` (Python)
- Usually synced with NTP
- Measured in seconds/milliseconds since epoch (Jan 1, 1970)

</v-clicks>

</div>

<v-click>

<div class="mt-6 p-4 bg-red-900/30 rounded-lg">

**⚠️ Danger: Time-of-day clocks can JUMP**

- NTP sync causes sudden adjustments
- Clock can move **backward**!
- Leap seconds cause weird behavior

</div>

</v-click>

<!--
Time-of-day clocks return "wall clock" time. What you'd see on a clock.

[click] In Java: System.currentTimeMillis.

[click] In JavaScript: Date.now.

[click] In Python: time.time.

[click] Usually synced with NTP.

[click] Measured in milliseconds since January 1, 1970.

[click] But here's the danger: time-of-day clocks can JUMP. NTP adjusts your clock. It can move backward! Imagine measuring duration: start at 12:00:00, NTP adjusts, end at 11:59:59. Duration: negative one second? Never use time-of-day clocks for duration!
-->

---

# Monotonic Clocks

<div class="mt-8">

**Designed for measuring elapsed time**

<v-clicks>

- `System.nanoTime()` (Java)
- `performance.now()` (JavaScript)
- `time.monotonic()` (Python)
- **Guaranteed to always move forward**
- Not tied to wall-clock time

</v-clicks>

</div>

<v-click>

<div class="mt-6 grid grid-cols-2 gap-4">
<div class="p-4 bg-green-900/30 rounded-lg">

**✅ Good for:**
- Measuring duration
- Timeouts
- Rate limiting
- Performance benchmarks

</div>
<div class="p-4 bg-red-900/30 rounded-lg">

**❌ Bad for:**
- Timestamps (no meaning across machines)
- Scheduling at specific times
- Anything needing real-world time

</div>
</div>

</v-click>

<!--
Monotonic clocks are different. Designed for measuring elapsed time.

[click] In Java: System.nanoTime.

[click] In JavaScript: performance.now.

[click] In Python: time.monotonic.

[click] Guaranteed to always move forward.

[click] Not tied to wall-clock time, only differences matter.

[click] Good for: measuring duration, timeouts, rate limiting, benchmarks. Bad for: timestamps, scheduling specific times, anything needing real-world time. The value has no meaning across machines.
-->

---

# Clocks in Practice: Monitor and Evict

<div class="mt-8">

**If your software relies on clocks, you MUST monitor them**

<v-clicks>

- Compare node clocks against NTP servers regularly
- Track drift over time
- **If a node drifts too far → declare it dead**
- Remove from cluster before it causes damage

</v-clicks>

</div>

<v-click>

<div class="mt-6 p-4 bg-yellow-900/30 rounded-lg">

**The "recent" problem**

What does "recent" mean? (e.g., "show posts from the last 5 minutes")

It depends on the local time-of-day clock... which may be **wrong**!

A node with a drifted clock might serve stale data thinking it's fresh.

</div>

</v-click>

<!--
Practical advice: if your system relies on clocks, you MUST monitor them.

[click] Compare node clocks against NTP servers regularly.

[click] Track drift over time.

[click] If a node drifts too far, declare it dead.

[click] Remove from cluster before it causes damage.

[click] Here's a tricky one: the "recent" problem. What does "recent" mean? "Show posts from the last 5 minutes." It depends on the local clock. Which may be wrong! A node with a drifted clock might serve stale data thinking it's fresh.
-->

---

# Confidence Intervals & TrueTime

<div class="mt-8">

**Problem:** We can't know the exact time, only an approximation

<v-clicks>

- NTP gives time ± some error margin
- Instead of a point in time, think of a **range**
- "The time is somewhere between [earliest, latest]"

</v-clicks>

</div>

<v-click>

<div class="mt-6 p-4 bg-blue-900/30 rounded-lg">

**Google Spanner's TrueTime API**

```
TrueTime.now() → [earliest, latest]
```

- Returns a confidence interval, not a single timestamp
- Spanner **waits out the uncertainty** before committing
- If intervals of two events don't overlap → guaranteed ordering
- Only Spanner implements this (requires GPS + atomic clocks in datacenters)

</div>

</v-click>

<!--
Here's a sophisticated approach: confidence intervals.

We can't know the exact time, only an approximation.

[click] NTP gives time plus or minus some error margin.

[click] Instead of a point in time, think of a range.

[click] "The time is somewhere between earliest and latest."

[click] Google Spanner's TrueTime API does exactly this. Returns a confidence interval, not a single timestamp. Before committing, Spanner waits out the uncertainty. If intervals don't overlap, you have guaranteed ordering.

But only Spanner does this. Requires GPS and atomic clocks in every datacenter. For everyone else, timestamps across machines are unreliable.
-->

---

# Clock Types: Summary

<div class="mt-8">

```
┌─────────────────────────┬────────────────────────┬────────────────────────┐
│                         │   TIME-OF-DAY CLOCK    │    MONOTONIC CLOCK     │
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ Purpose                 │ "What time is it?"     │ "How long did it take?"│
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ Can jump?               │ YES (NTP adjustments)  │ NO (always forward)    │
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ Comparable across       │ Yes (if synced)        │ NO                     │
│ machines?               │                        │                        │
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ Use for timestamps?     │ ⚠️  With caution       │ ❌ Never               │
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ Use for duration?       │ ❌ Never               │ ✅ Yes                 │
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ Precision               │ Milliseconds           │ Nanoseconds            │
└─────────────────────────┴────────────────────────┴────────────────────────┘
```

</div>

<!--
Here's a quick reference comparing the two clock types.

Time-of-day: answers "what time is it?" Can jump due to NTP. Comparable across machines if synced. Use for timestamps with caution. Never use for duration.

Monotonic: answers "how long did it take?" Never jumps. Not comparable across machines. Never use for timestamps. Always use for duration.

Know which clock you're using and why!
-->

---

# Key Takeaways

<div class="grid grid-cols-2 gap-8 mt-8">
<div>

**Networks lie**

<v-clicks>

- Requests can fail silently
- You can't tell failure from slowness
- Timeouts are guesses, not guarantees
- Design for network partitions

</v-clicks>

</div>
<div>

**Clocks lie**

<v-clicks>

- Clocks drift without synchronization
- NTP helps but isn't perfect
- Time-of-day clocks can jump backward
- Use monotonic clocks for duration

</v-clicks>

</div>
</div>

<v-click>

<div class="mt-8 p-4 bg-blue-900/30 rounded-lg text-center text-xl">

**Assume failure. Design for resilience. Trust nothing.**

</div>

</v-click>

<!--
Key takeaways.

Networks lie.

[click] Requests can fail silently.

[click] You can't tell failure from slowness.

[click] Timeouts are guesses, not guarantees.

[click] Design for network partitions.

Clocks lie.

[click] Clocks drift without synchronization.

[click] NTP helps but isn't perfect.

[click] Time-of-day clocks can jump backward.

[click] Use monotonic clocks for duration.

[click] The overarching message: Assume failure. Design for resilience. Trust nothing.
-->

---

# Discussion

<div class="mt-12 text-center">

<div class="text-6xl mb-8">🤔</div>

**Questions for the group:**

<v-clicks>

1. Have you experienced mysterious production issues that turned out to be clock-related?

2. How does your team handle network timeouts today?

3. What's the longest outage you've seen from a "simple" infrastructure failure?

</v-clicks>

</div>

<!--
Let's open it up for discussion.

[click] Have you experienced mysterious production issues that turned out to be clock-related?

[click] How does your team handle network timeouts today?

[click] What's the longest outage you've seen from a "simple" infrastructure failure?

Feel free to share war stories. We've all been burned by these issues.
-->
