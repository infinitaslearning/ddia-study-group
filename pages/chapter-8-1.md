<!-- # The Trouble with Distributed Systems -->

<div class="text-center mt-20">
<div class="text-6xl mb-8">💀</div>
<p class="text-2xl text-gray-400">"Everything that can go wrong, will go wrong"</p>
<p class="text-lg text-gray-500 mt-4">Networks, Clocks & Partial Failures</p>
</div>

<!--

This is the most pessimistic chapter in the book.

We'll cover unreliable networks, unreliable clocks, and why partial failures make everything hard.
-->

---

# A Pessimistic Worldview

<div class="grid grid-cols-2 gap-8 mt-8">
   <div>

<v-clicks>

- "In distributed systems: **suspicion, pessimism, and paranoia pay off**"
- Assume components can be slow or faulty:
  - the network
  - the clocks
  - other nodes
- **Partial failures** are the norm, not the exception

</v-clicks>

   </div>
<div class="flex items-center justify-center">


</div>
</div>

<!--
The key mindset for this chapter: assume everything is broken.

[click] In distributed systems, suspicion, pessimism, and paranoia pay off.

[click] Assume components can be slow or faulty: the network, clocks, other nodes.

[click] Partial failures are the norm. Unlike a single computer where things work or don't, distributed systems can be partially working and partially failing at the same time.
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

   "No response ≠ failed request"
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

[click] And here's the thing — from the sender's perspective, you often can't tell which one happened. All you know is: no response.
-->

---

# Ambiguous Outcomes

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

A positive acknowledgment can reduce uncertainty (but lack of one is still ambiguous)

</div>

</v-click>

<!--
This is an ambiguous-outcome problem.

The sender cannot distinguish between:

[click] The request was lost.

[click] The remote node is down.

[click] The remote node is just slow.

[click] The response was lost.

[click] They all look the same: silence. No response. You wait, then timeout.

A positive acknowledgment can confirm success, but lack of one remains ambiguous.
-->


---

# Timeouts: The Default Failure Detector

<div class="mt-8">

**Problem:** How long do you wait before giving up?

<v-clicks>

- **Too short:** False positives - suspect slow nodes as failed
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
Timeouts are the default tool for detecting failures. How long do you wait before giving up?

[click] Too short: you get false positives. You treat “slow” as “dead” and trigger retries/failovers.

[click] Too long: users wait forever. Your system feels hung.

[click] Let me break down the specific risks. On the left: cascading failures, unnecessary failovers, retry storms. On the right: poor user experience, slow failure detection, resource starvation.

In practice: pick a timeout based on what you observe in real response times, then revisit it as conditions change.
-->

---

# Why Delays Are Unbounded

<div class="text-center mt-4 text-xl">

**Async networks provide no reliable maximum delay bound**

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

**Queueing can happen at many steps:** network switches, routers, OS buffers, hypervisors, receiving application

</div>

</v-click>

<!--
The internet is a packet-switched network. There are no hard guarantees about timing.

[click] Look at the journey of a packet. It goes through your app, OS buffer, network card, switch queues, routers, more switches, back through OS buffers, finally to the receiving app.

[click] Queueing can happen at many steps. Network switches, routers, OS buffers, hypervisors, the receiving application. These delays are variable and unpredictable — which is why picking a "correct" timeout is impossible.
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

**Why do real-time apps often use UDP instead of TCP?**

<v-clicks>

- TCP guarantees delivery by **retransmitting** lost packets
- For real-time media: an old packet is often **useless**
- Better to skip the frame than delay the stream
- UDP: "Fire and forget" - no retransmission, no ordering, no retransmission-induced delay

</v-clicks>

</div>

<v-click>

<div class="mt-6 p-4 bg-gray-200 rounded-lg">

**Trade-off:**
- TCP: Reliable but unpredictable latency
- UDP: Predictable latency but potential data loss

Choose based on what matters more to your application.

</div>

</v-click>

<!--
Why do real-time apps often use UDP instead of TCP?

[click] TCP guarantees delivery by retransmitting lost packets.

[click] For real-time media, a late packet is often useless. The conversation/game has moved on.

[click] Better to skip the frame than delay the stream.

[click] UDP is "fire and forget" at the transport layer: no retransmission, no in-order delivery. Network delay still exists — you’re just not adding extra delay by waiting for missing packets.

[click] It's a trade-off. TCP: reliable but latency can spike under loss. UDP: lower latency variance, but you must tolerate loss.
-->

---

# Dynamic Timeouts

<div class="mt-8">

**Static timeouts are suboptimal - conditions change!**

<v-click>

**Solution: Adapt based on observed behavior**

</v-click>

<v-clicks>

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
- Reserved capacity end-to-end
- More predictable latency (bounded delay in principle)
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

Packet switching trades strict guarantees for flexibility and efficiency

</div>

</v-click>

<!--
Why is the internet unreliable? Compare it to phone networks.

Phone networks are circuit-switched.

[click] Dedicated bandwidth per call.

[click] Reserved capacity end-to-end.

[click] More predictable latency - bounded delay in principle.

[click] But inefficient: bandwidth wasted when you're silent.

[click] Now the internet - packet-switched. Share bandwidth dynamically.

[click] No guarantees - best effort only.

[click] Variable latency - unbounded delay.

[click] But efficient: only use what you need.

[click] Packet switching trades strict guarantees for flexibility and efficiency — great for utilization, but it means delays are variable.
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

<div class="mt-6 p-4 bg-gray-200 rounded-lg">

**Clock Drift**

```
Typical drift: ~1–10 seconds per day (commodity hardware)
Worst case:   tens of seconds per day (temperature, cheap crystal, aging)

Over a month: Could be off by minutes!
```

</div>

</v-click>

<!--
Every computer has a quartz crystal oscillator.

[click] It vibrates at a known frequency.

[click] Count oscillations, track time.

[click] Simple, but crystals aren't perfect.

[click] The problem: they drift!

[click] Think of it like a cheap wristwatch — runs a bit fast or slow. A decent clock might drift a few seconds per day; bad hardware or bad conditions can be much worse. Leave a server running for a month without sync? Minutes off.
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

[click] But there are problems. Network delays vary. Leap seconds cause jumps. Misconfigured servers exist. On a LAN you might get millisecond-ish accuracy; across the internet it can be tens of milliseconds (or worse).
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
- Not stepped by NTP (shouldn't go backward)
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

[click] Not stepped by NTP, so it shouldn’t jump backwards.

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
- **If a node drifts too far → quarantine it**
- Remove it from serving before it violates assumptions

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
If your system relies on clocks, monitor them.

[click] Compare node clocks against NTP servers regularly.

[click] Track drift over time.

[click] If a node drifts too far, quarantine it.

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
- If intervals of two events don't overlap → you can infer ordering
- Canonical example: Spanner (requires GPS + atomic clocks in datacenters)

</div>

</v-click>

<!--
So we can't know the exact time — only an approximation.

[click] NTP gives you time plus or minus some error margin.

[click] Instead of a single point, think of it as a range.

[click] "The time is somewhere between earliest and latest."

[click] Google Spanner's TrueTime API exposes this. Returns a confidence interval instead of a timestamp. Spanner waits out the uncertainty before committing — if intervals don't overlap, you can infer a real-time order.

The catch? This requires specialized time infrastructure (GPS + atomic clocks) and careful engineering. For everyone else, timestamps across machines remain unreliable.
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
│ Can jump?               │ YES (NTP adjustments)  │ NO (not stepped by NTP)│
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ Comparable across       │ Yes (if synced)        │ NO                     │
│ machines?               │                        │                        │
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ Use for timestamps?     │ ⚠︎  With caution        │ ⨯ Never                │
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ Use for duration?       │ ⨯ Never                │ ✔︎ Yes                  │
├─────────────────────────┼────────────────────────┼────────────────────────┤
│ Resolution (API)        │ ms-scale               │ high-resolution        │
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

# Discussion

<div class="mt-12 text-center">

<div class="text-6xl mb-8">🤔</div>

**Let's discuss:**

<v-clicks>

1. Anyone debugged a weird production issue that ended up being clock drift? What was it?

2. What timeout values does your team use? How did you arrive at them?

3. What's your favorite "one typo took down production" story?

</v-clicks>

</div>

<!--
Let's open it up for discussion.

[click] Have you experienced mysterious production issues that turned out to be clock-related?

[click] How does your team handle network timeouts today?

[click] What's the longest outage you've seen from a "simple" infrastructure failure?

Feel free to share war stories. We've all been burned by these issues.
-->
