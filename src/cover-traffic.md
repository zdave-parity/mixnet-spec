# Cover traffic

There are two types of cover traffic: drop and loop. They differ in two ways:

- Destination selection for a drop cover packet is the same as for a request: a random mixnode,
  excluding the source node and, if there is exactly one, its connected gateway mixnode. Loop cover
  packets are always sent from a node to itself.
- Drop cover packets are replaced by packets from the [request/reply
  queue](./requests-and-replies.md#packet-queues) for the session. Loop cover packets are never
  replaced.

Nodes should dispatch drop and loop cover packets in each active session according to Poisson
processes. The average drop/loop rates should be the same for all mixnodes in a session, and for
all non-mixnodes in a session; see the [parameters chapter](./parameters.md). Note that the rates
should be halved in some phases; see the [sessions chapter](./sessions.md#phases).

TODO maybe be explicit if `Node` is `mixnet-node` or if it includes `non-mixnet-nodes`.
