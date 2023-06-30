# Message fragmentation

Request and reply messages are split into 2,048-byte fragments. Each fragment of a message is
wrapped in a [Sphinx packet](./sphinx.md) and sent along a different route to the destination. The
destination node is responsible for collecting the fragments together and reassembling the message.

## Fragment structure

// TODO it is not exactly clear where this fragment is when looking at `sphinx`. IIUC sphinx payload contains
// the fragment. (means that some of the point I raised in `sphinx.md` can just refer to this md.
The 2,048 bytes in a fragment are split as follows:

| Bytes    | Field                   | Description                                                    |
|----------|-------------------------|----------------------------------------------------------------|
| 0..15    | `message_id`            | The same for all fragments of a message                        |
| 16..17   | `num_fragments_minus_1` | Little-endian number of fragments minus one                    |
// TODO theorical max size of a full message (2^16 + 1) * fragement payload size.
// TODO with two byte of size, is it really worth it to minus 1, I mean it prevents size 0, but size 0 is not
// that problematic, it would just mean empty message.
| 18..19   | `fragment_index`        | Little endian index of this fragment                           |
| 20..21   | `data_size`             | Little-endian number of bytes of message data in this fragment |
| 22       | `num_surbs`             | Number of SURBs in this fragment                               |
| 23..2047 | `payload`               | Message data and SURBs                                         |
// TODO explicitely state if surbs are before or after message.

`payload` has `data_size` bytes of message data at the start, and `num_surbs`
[SURBs](./sphinx.md#surbs) tightly packed at the end. Any unused bytes in the middle should be
written as 0 by message senders, but ignored by receivers.

## Reassembly

Received fragments should be grouped by `message_id`. Once a full set of fragments for a
`message_id` has been received, the message data and SURBs from each fragment should be
concatenated, in index order, to give the original message and an array of SURBs that can be used
to reply. The fragments should then be discarded; it is important that no SURB is used more than
once.

The session and type (request/reply) of a message should be inferred from the last received
fragment, and the message data and SURBs [handled appropriately](./requests-and-replies.md). In the
case of a reply message, the corresponding request message ID should also be inferred from the last
received packet: request message IDs should be stored alongside payload encryption keys in the SURB
keystore.

Note that:

- Malicious nodes may send fragments with the same `message_id` but differing
  `num_fragments_minus_1`. Simply discarding all fragments with a different `num_fragments_minus_1`
  to the first-received fragment is acceptable.
- Nodes should enforce a limit on the number of fragments per message (see the [parameters
  chapter](./parameters.md)), by simply discarding fragments with a too-large
  `num_fragments_minus_1`.
// TODO should common parameter like this be part of the runtime api?
- Multiple fragments with the same `message_id` and `fragment_index` may be received. It is
  recommended that only one be kept.
// TODO maybe also acceptable to drop all if they differs in content.
// Explicitelly this means that a packet can be send twice? Or we store only per message_id (in this
// case the id may be too small to prevent collision?).
- Invalid fragments, with `fragment_index > num_fragments_minus_1` or `(data_size + (num_surbs *
  222)) > 2025`, should be discarded.

It is expected that nodes will only keep a limited number of fragments, for the most recently seen
`message_id`s.

## Construction

Message IDs should be randomly generated. When retransmitting a message, the message ID should be
reused. If a new destination is chosen for retransmission however, or if the retransmission is in a
different session, a new message ID should be generated.

// TODO 128 bit of random is same as a uuid v4 so should prevent collision.

There are essentially no rules on how message data and SURBs may be split amongst fragments; the
only constraints are:

- The message data and SURBs assigned to each fragment must fit.
- The fragments-per-message limit must not be exceeded.

It _is_ important however that the same split is used if a message is retransmitted with the same
`message_id`.

When retransmitting a fragment, exactly the same message data should be sent, but new SURBs must be
generated: no SURB should ever be sent twice. When generating SURBs for a request, the message ID
should be stored in the SURB keystore alongside the payload encryption keys.
