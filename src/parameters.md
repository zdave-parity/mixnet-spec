# Parameters

All nodes in a Substrate Mix Network should agree on the following parameters:

- Mean forwarding delay. This is the average artificial packet delay at [each
  hop](./sphinx.md#forward-actions) along a route.
- Mean extrinsic delay. This is the average artificial delay between receipt of a
  [`SubmitExtrinsic`](./requests-and-replies.md#submitextrinsic) request and import of the attached
  extrinsic.
- Maximum fragments per message. See the [message fragmentation
  chapter](./message-fragmentation.md#reassembly).
- Maximum length of a mixnode's [request/reply queue](./requests-and-replies.md#packet-queues).
  Note that in sessions where a node is not a mixnode, it is free to choose the queue bound itself.
- Reply cooldown. If a node receives two requests with the same message ID within this time period,
  it should not reply to both of them. See the [requests and replies
  chapter](./requests-and-replies.md#request-handling).
- Average cover traffic rates. There are four independent rates: mixnode loop, mixnode drop,
  non-mixnode loop, and non-mixnode drop. See the [cover traffic chapter](./cover-traffic.md).
