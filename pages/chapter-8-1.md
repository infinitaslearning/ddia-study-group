# The Trouble with Distributed Systems

### Intro

* Consider everything that can go wrong.
* It’s a **pessimistic** and **depressing** overview of things that may go wrong in a distributed system.
* Distributed systems suffer nondeterministic and partial failures, which is why it’s hard to work with.
* In distributed systems, suspicion, pessimism, and paranoia pay of.



---

# Networks

are unstable and unreliable
	we don’t know when the recipient node or network fails, the error is the same so the solution is a timeout
systems should be able to handle network errors and recover from them




---

# timeouts
async networks have unbounded delays, there is no guarantee on the max response time. 

---

# congestion and queueing

---

packet loss
	udp for streaming

---

dynamic timeout  based on response times over an extended period

---

async: TCP variable bandwith

---

sync: phone network, fixed bandwidth

---

qos, prioritization, and scheduling of packages and admission control (rate-limiting sender) can emulate circuit switching

---

Clock and timing issues

---

quarz crystal oscillator

---

Network Time Protocl NTP. allow machines to adjust time  from servers connected to a more accurate source such as GPS receiver

---

Time-of-day clocks
it returns the current date and time
usually synchronized with NTP

---

Monotonic
suitable to measure duration
useful for timeout
always move forward
NTP can udjust the the clock rate to be speeded up or slowed down but can’t move it backward or forward
they can measure time intervals in mircroseconds or less

---

