# Requests and replies

Request and reply messages are SCALE encoded. Requests have the following type:

    enum Request {
        #[codec(index = 1)]
        SubmitExtrinsic(Extrinsic),
    }

Where `Extrinsic` is the extrinsic type of the blockchain.

Replies have types of the form `Result<T, RemoteErr>`, where `T` depends on the request, and
`RemoteErr` is defined as follows:

    enum RemoteErr {
        Other(String),
        Decode(String),
    }

`Decode` means the node failed to decode the request. `Other` means the node encountered some other
error. In both cases the `String` is a description of the error, suitable for presenting to the
user.

Note that nodes may simply ignore malformed requests, instead of responding with `Err(Decode)`.

## `SubmitExtrinsic`

A `SubmitExtrinsic` request can be sent to anonymously submit an extrinsic to the blockchain. The
reply type is `Result<(), RemoteErr>`. `Ok(())` means the receiving node successfully imported the
extrinsic into its transaction pool. It does _not_ mean that the extrinsic was included in the
blockchain.

After receiving a `SubmitExtrinsic` request, a node should wait before attempting to import the
extrinsic. The delay should be determined from the request message ID as follows:

    seed = blake2b("submit-extrn-dly", 0, message_id)[..16]
    delay = exp_random(seed) * mean_delay

Where `mean_delay` is the mean extrinsic delay; see the [parameters chapter](./parameters.md).

## Packet queues

Nodes should maintain a bounded request/reply packet queue for each session. Packets from a
session's queue should be dispatched in place of [drop cover packets](./cover-traffic.md) for the
session (provided the [current phase](./sessions.md#phases) permits request/reply packets to be
sent). The queue bound should be the same for all mixnodes in a session; see the [parameters
chapter](./parameters.md).

Reply packets should simply be dropped if they cannot be pushed onto the request/reply queue. It is
recommended however that nodes use additional queues and/or back-pressure to avoid dropping request
packets.

## Request sending

The session for a request should be chosen according to the table in the [sessions
chapter](./sessions.md#phases). The destination node should be uniformly randomly picked from the
mixnodes for the session, excluding the source node and, if there is exactly one, its connected
gateway mixnode. The same session and destination should be used for retransmission, unless:

- The phase changes such that request packets may no longer be sent in the chosen session.
- The destination fails to reply too many times.

In these cases, a new session and destination should be selected.

The request message should be [split into fragments](./message-fragmentation.md#construction),
which should then be [wrapped in Sphinx packets](./sphinx.md#packet-construction) and pushed onto
the session's request/reply queue. The route for each packet should be generated independently as
described in the [topology chapter](./topology.md#route-generation).

Multiple copies of each fragment may be sent to improve the chance of success. Each copy of a
fragment should contain different SURBs, and should be sent along a different route to the
destination.

### Retransmission

The round-trip time for a request can be conservatively estimated as follows:

    (s, n, m, r) = if request_period > reply_period {
        (request_period, request_len, reply_len, reply_period / request_period)
    } else {
        (reply_period, reply_len, request_len, request_period / reply_period)
    }
    queue_delay = s * (4.92582 + (3.87809 * sqrt(n + (r * r * r * m))) + n + (r * m))

    net_delay = per_hop_net_delay * num_hops

    rtt = forwarding_delay + queue_delay + net_delay + handling_delay

Where:

- `request_period` and `reply_period` are the average periods between drop cover packet dispatches
  for the session from the source node and the destination node respectively. Note that:
  - All mixnodes in a session should dispatch drop cover packets at the same (average) rate; see
    the [parameters chapter](./parameters.md).
  - From the perspective of the nodes, the session phase may change at any time. As such, it should
    be assumed, for a conservative estimate, regardless of the current phase, that traffic will be
    sent at half rate in all sessions.
- `request_len` is the number of request/reply packets queued ahead of the last request packet,
  plus one.
- `reply_len` is the maximum length of the destination node's request/reply queue. Note that all
  mixnodes in a session should have the same queue bound; see the [parameters
  chapter](./parameters.md).
- The `queue_delay` expression is an approximation of the 99.995th percentile of the sum of two
  independent gamma-distributed random variables with different scales (`s`, `s * r`) and shapes
  (`n`, `m`).
- `per_hop_net_delay` is a conservative estimate of the network (and processing) delay per hop.
- `num_hops` is the maximum number of hops for any of the fragments to reach the destination, plus
  the maximum number of hops for any of the SURBs to come back. The number of hops in a route is
  the number of nodes (including both source and destination) minus one.
- `forwarding_delay` is the maximum total [forwarding delay](./sphinx.md#forward-actions) for any
  request fragment, plus the maximum total forwarding delay for any SURB.
- `handling_delay` is a conservative estimate of the time taken to handle the request at the
  destination and post the reply. In the case of a [`SubmitExtrinsic`](#submitextrinsic) request
  for example, this should include the artifical extrinsic delay.

If a complete reply message is not received within this time, the request message may be
retransmitted. When retransmitting, different packet routes should be used, and new SURBs should be
generated.

## Request handling

Sending a reply message is similar to sending a request message. The message is [split into
fragments](./message-fragmentation.md#construction), which are then wrapped in Sphinx packets
([using the SURBs](./sphinx.md#surb-use) attached to the request message) and pushed onto the
session's request/reply queue. Multiple copies of each fragment may be sent, provided enough SURBs
were attached to the request message, and there is enough space in the request/reply queue.

To avoid handling the same request multiple times, reply messages should be cached by request
message ID. When sending a cached reply, the original reply message ID should be reused, but the
Sphinx packets should be constructed from different SURBs (eg the ones attached to the triggering
request message).

If two request messages with the same ID are received in short succession (as determined by the
reply cooldown [parameter](./parameters.md)), the second message should be assumed to be a copy
sent at the same time as the first, rather than a retransmission, and should be ignored.

## Reply handling

The request message ID corresponding to a received reply message should be inferred from the last
received packet; the SURB keystore stores request message IDs alongside payload encryption keys.
The request to which the message is a reply should be determined from this ID. If the ID is no
longer recognised, the reply message should simply be dropped.

Any SURBs attached to a reply message should be ignored.
