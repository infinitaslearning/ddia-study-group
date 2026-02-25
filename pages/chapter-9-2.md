# Consistency and Consensus

Part 2

---

# Total Order Broadcast

<v-clicks>

- A protocol for exchanging messages between nodes
- Requires two properties:
- **Reliable delivery**: no messages are lost; if a message is delivered to one node, it is delivered to all nodes (even if a node or the network is faulty)
- **Totally ordered delivery**: messages are delivered to every node in the same order

</v-clicks>

---

# Total Order Broadcast: where it shows up

<v-clicks>

- Consensus services (e.g., ZooKeeper)
- Database replication
- Serializable transactions
- Lock services with fencing tokens (see The leader and the lock)
- Linearizable storage (e.g., ensuring only one user can claim a unique username)

</v-clicks>
