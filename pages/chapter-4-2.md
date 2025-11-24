---
class: text-center
layout: cover
---
# Part 2: Modes of Data flow

---

1. Via Database
2. Via service calls
3. Via async message passing

---

## Dataflow Through Databases

> "sending a message to your future self"

### Different values written at different times

<img src="../assets/chapter04/old-app-vs-new-app.jpg" style="display: block; margin: 2em auto 0; max-height: 400px; width: auto; max-width: 700px;"/>

---

## Dataflow Through Services: REST and RPC

---

## Web services

| **SOAP API** | **REST API** |
| -- | -- |
| Relies on SOAP (Simple Object Access Protocol) | Relies on REST (Representational State Transfer) architecture using HTTP. |
| Transports data in standard XML format.	Generally transports data in JSON. It is based on URI. | Because REST follows a stateless model, REST does not enforce message format as XML or JSON etc. |
| Because it is XML based and relies on SOAP, it works with WSDL| It works with GET, POST, PUT, DELETE |
| Works over HTTP, HTTPS, SMTP, XMPP| Works over HTTP and HTTPS |
| Highly structured/typed | Less structured -> less bulky data
Designed with large enterprise applications in mind	| Designed with mobile devices in mind |

---

## The problems with remote procedure calls (RPCs)

- Unpredictability
- Timeout
- Idempotency
- Latency is wildly variable
- Parameter encoding: large objects
- Datatype mismatch

> "appeal of REST is that it doesn’t try to hide the fact that it’s a network protocol"

> "RESTful API has other significant advantages: it is good for experimentation and debugging"

---

## Message-Passing Dataflow

---


---

## Distributed actor frameworks