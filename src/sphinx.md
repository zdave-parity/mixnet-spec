# Sphinx

Substrate Mix Networks use a packet format based on
[Sphinx](https://cypherpunks.ca/~iang/pubs/Sphinx_Oakland09.pdf).

## Packet structure

All packets are the same size: 2,252 bytes. These bytes are split as follows:

| Bytes     | Field       | Description                                           |
|-----------|-------------|-------------------------------------------------------|
| 0..31     | `kx_public` | X25519 public key, &alpha; in the Sphinx paper        |
| 32..47    | `mac`       | MAC, &gamma; in the Sphinx paper                      |
| 48..187   | `actions`   | Encrypted routing actions, &beta; in the Sphinx paper |
| 188..2251 | `payload`   | Encrypted payload, &delta; in the Sphinx paper        |

The `kx_public`, `mac`, and `actions` fields together form the header.

## Key exchange

The X25519 public key `kx_public` is combined with the receiving node's X25519 secret key for the
session `recv_kx_secret` to produce a 32-byte shared secret `kx_shared_secret`:

    kx_shared_secret = Curve25519(recv_kx_secret, kx_public)

The same shared secret can be derived by combining the secret key corresponding to `kx_public`,
`kx_secret`, with the receiving node's public key `recv_kx_public`:

    kx_shared_secret = Curve25519(kx_secret, recv_kx_public)

Note that `kx_public` bears no relation to the source node's X25519 key pair for the session; the
corresponding secret key is [randomly generated per packet](#key-generation).

## Secret derivation

From the 32-byte shared secret, a number of other secrets are derived:

- A 32-byte X25519 blinding factor `kx_blinding_factor`.
- A 16-byte MAC key `mac_key`.
- A 32-byte routing actions encryption key `actions_encryption_key`.
- A 16-byte forwarding delay seed `delay_seed`.
- A 192-byte payload encryption key `payload_encryption_key`.

These are derived as follows:

    kx_blinding_factor = clamp_scalar(
        blake2b("sphinx-blind-fac", 0, kx_public . kx_shared_secret)[..32])

    mac_key . actions_encryption_key . delay_seed =
        blake2b("sphinx-small-d-s", 0, kx_shared_secret)

    payload_encryption_key =
        blake2b("sphinx-pl-en-key", 0, kx_shared_secret) .
        blake2b("sphinx-pl-en-key", 1, kx_shared_secret) .
        blake2b("sphinx-pl-en-key", 2, kx_shared_secret)

## MAC verification

`mac` should equal the BLAKE2b hash of `actions` computed with the key `mac_key`.

The receiving node determines the packet's session by which X25519 secret key results in the hash
equalling `mac`. If neither the previous nor current session key results in a hash equalling `mac`,
the packet is simply dropped.

## Routing actions

The routing actions in `actions` dictate how the packet should be routed. There should be a
"forward" action for each intermediate node, followed by a single "deliver" action for the final
node.

Actions are two-byte little-endian unsigned integers, with some actions being followed by
additional data:

| Action   | Description                                | Additional data              |
|----------|--------------------------------------------|------------------------------|
| < 0xff00 | Forward to the mixnode with this index     | 16-byte MAC                  |
| 0xff00   | Forward to the node with the given peer ID | 32-byte peer ID, 16-byte MAC |
| 0xff01   | Deliver request packet                     | None                         |
| 0xff02   | Deliver reply packet                       | 16-byte SURB ID              |
| 0xff03   | Deliver cover packet                       | None                         |
| 0xff04   | Deliver cover packet with ID               | 16-byte cover ID             |
| > 0xff04 | Invalid                                    | N/A                          |

Actions are tightly packed, with the first two bytes of `actions` giving the first action.

`actions` is encrypted with the ChaCha20 stream cipher, keyed by `actions_encryption_key`, using a
nonce of zero. Actions past the first are encrypted further with different keys; only the first
action can be fully decrypted by the receiving node.

### Forward actions

If the first action is a forward action, and the receiving node is a mixnode in the session, it
should attempt to forward the packet to the specified node. If the receiving node is not a mixnode,
the packet should be discarded. Before forwarding, the packet should be artificially delayed, and
transformed.

The artificial delay should be calculated as `exp_random(delay_seed) * mean_delay`, where
`mean_delay` is the mean forwarding delay (see the [parameters chapter](./parameters.md)).

The packet should be transformed as follows:

| Field       | Transformation                                              |
|-------------|-------------------------------------------------------------|
| `kx_public` | Replace with `Curve25519(kx_blinding_factor, kx_public)`    |
| `mac`       | Replace with the MAC following the forward action           |
| `actions`   | Extend with zero bytes, decrypt, then drop the first action |
| `payload`   | Decrypt using `payload_encryption_key`                      |

Note that:

- The number of zero bytes appended to `actions` should match the length of the first action (2
  plus the length of any additional data); after dropping the first action the total length should
  match the original length prior to extension.
- Decryption of the extended `actions` simply means XORing with the ChaCha20 keystream derived from
  `actions_encryption_key`.

### Deliver actions

If the first action is a deliver action, the packet is destined for the receiving node.

If the packet is a cover packet, it should simply be discarded.

If the packet is a request packet, and the receiving node is a mixnode in the session, the payload
should be decrypted using `payload_encryption_key`. If the receiving node is _not_ a mixnode, the
packet should be discarded.

If the packet is a reply packet, the receiving node should lookup and remove the payload encryption
keys corresponding to the SURB ID from its SURB keystore. If the keys are not found, the packet
should be discarded. Otherwise, the keys should be used, one at a time in reverse order, to
_encrypt_ the payload. See [the SURBs section below](#surbs) for more on SURBs.

After decryption/encryption of a request/reply payload, if the last 16 bytes of the payload are 0,
the rest of the payload should be handled as a [message
fragment](./message-fragmentation.md#fragment-structure). If the last 16 bytes are not all 0, the
packet is invalid and should be discarded.

### Invalid actions

If the first action is invalid, the receiving node should discard the packet.

## Payload encryption

The LIONESS cipher, instantiated with BLAKE2b as the hash function and ChaCha20 as the stream
cipher, is used for payload encryption. LIONESS is described in detail in [Two Practical and
Provably Secure Block Ciphers: BEAR and
LION](https://www.cl.cam.ac.uk/~rja14/Papers/bear-lion.pdf).

Encryption keys, such as `payload_encryption_key`, are the concatenation of K<sub>1</sub>,
K<sub>2</sub>, K<sub>3</sub>, and K<sub>4</sub> (see the linked paper), with K<sub>1</sub> and
K<sub>3</sub> being 32 bytes, and K<sub>2</sub> and K<sub>4</sub> being 64 bytes.

The ChaCha20 stream cipher is initialised with a zero nonce.

The `lioness` crate on `crates.io` provides a compatible implementation.

## Replay filtering

If a node receives multiple packets which have the same `kx_shared_secret`, it must avoid
forwarding or delivering more than one of them. Note that:

- When a secret key is discarded (eg due to a session switch), all shared secrets which were
  derived from the secret key may be forgotten.
- If a packet is discarded before being forwarded or delivered, its shared secret need not be
  remembered.
- It is not necessary to record reply packet shared secrets, as the lookup in the SURB keystore
  will fail for replayed SURBs. Further, as cover packets are simply discarded, and request and
  forward packets are not accepted by non-mixnodes, if a node is never a mixnode, it need not do
  any explicit replay filtering.
- It is expected that nodes will use a probabilistic data structure, such as a bloom filter, to
  record which shared secrets have been derived from each secret key. The false-negative rate of
  the data structure should be zero. The false-positive rate should be below 1% (ie there should be
  at most 1% packet loss caused by faulty filtering). Shared secret hashes should be cryptographic,
  and keyed to avoid DoS attacks.

## Packet construction

This section covers the construction of request and cover packets. Reply packet construction is
slightly different as it is split into two parts, which take place on different nodes: SURB
construction (receiving node) and SURB use (sending node). This is covered in the [SURBs section
below](#surbs).

### Key generation

The first step in constructing a Sphinx packet is key generation. An X25519 key pair should be
generated as follows:

    kx_secret = clamp_scalar(random())
    kx_public = Curve25519(kx_secret, 9)

Where:

- `random()` generates 32 random bytes.
- `9`, the X25519 base point, is little-endian encoded.

### Shared secret computation

The shared secrets can then be computed incrementally:

    kx_secrets[0] = kx_secret
    kx_shared_secrets[i] = Curve25519(kx_secrets[i], recv_kx_publics[i])
    kx_publics[i] = Curve25519(kx_secrets[i], 9)
    kx_secrets[i + 1] = (kx_blinding_factors[i] * kx_secrets[i]) % order

Where:

- `recv_kx_publics[i]` is the X25519 public key of the `i`th node in the route (excluding the
  source node).
- `kx_blinding_factors[i]` is computed from `kx_publics[i]` and `kx_shared_secrets[i]` as described
  [above](#secret-derivation).
- `order` is 2<sup>252</sup> + 27742317777372353535851937790883648493; the order of the group
  generated by the X25519 base point.

### Routing actions

Generating the encrypted routing actions (`actions`) is non-trivial; the MACs attached to the
forward actions depend on the "padding" (encrypted zeroes) the receiving nodes will see. One method
is as follows:

- Write the routing actions, unencrypted, into `actions`. Leave any MACs uninitialised. Fill any
  unused bytes with random data.
- Compute the padding that each node along the route will observe on packet arrival. The length at
  each node should match the total length of all earlier actions. The padding is generated by
  zero-extension and encryption at each node along the route; it can be determined from the action
  sizes and `kx_shared_secrets`.
- For each action, in reverse order:
  - Encrypt all bytes from the start of the action to the end of `actions`, using the encryption
    key for the node that will process the action.
  - Compute the MAC of the concatenation of the just-encrypted part of `actions` with the
    corresponding padding computed in the second step, using the MAC key for the node that will
    process the action. In the case of the first action, write the MAC to `mac`; for other actions,
    write it into `actions` immediately before the action.

Whichever method is used, it is important that any unused decrypted data the final node sees is
indistinguishable from random. If this is not the case, the final node may be able to infer the
route length. The above method achieves this by filling unused bytes with random data in the first
step.

Note that the 140-byte `actions` is just large enough to handle the [worst case
route](./topology.md#route-generation):

- 4 forward-to-mixnode actions (18 bytes each, 72 bytes total).
- One forward-to-peer-ID action (50 bytes).
- One deliver action with a 16-byte ID (18 bytes).

### Payload

Constructing `payload` is straightforward. For cover packets, simply fill `payload` with random
bytes. For request packets:

- Write zeroes to the last 16 bytes. Write the unencrypted [message
  fragment](./message-fragmentation.md#fragment-structure) to the other bytes.
- [Derive the payload encryption keys](#secret-derivation) for the nodes along the route from
  `kx_shared_secrets`.
- Encrypt `payload` using each encryption key, in reverse order (starting with the encryption key
  for the destination node).

## SURBs

A SURB (single-use reply block) can be used to send a reply packet to the node that generated it.
As the name suggests, each SURB should only be used once. SURBs are always 222 bytes, split as
follows:

| Bytes    | Field                 | Description                                            |
|----------|-----------------------|--------------------------------------------------------|
| 0..1     | `first_mixnode_index` | Little-endian index of the first mixnode in the route  |
| 2..189   | `header`              | Prefabricated Sphinx header                            |
| 190..221 | `shared_secret`       | Secret to derive the first payload encryption key from |

### SURB construction

Equipped with a [route](./topology.md#route-generation), a node can build a SURB as follows:

- Set `first_mixnode_index` to the index of the first mixnode in the route, excluding the source
  node.
- Build the Sphinx header `header` as [normal](#packet-construction), with the last action being a
  deliver reply action with a randomly generated SURB ID.
- Randomly generate `shared_secret`.
- [Derive the payload encryption keys](#secret-derivation) for the nodes along the route from
  `kx_shared_secrets`, excluding the source and destination nodes.
- Derive a payload encryption key from `shared_secret` in the same way.
- Store the payload encryption keys (the one derived from `shared_secret`, followed by the ones
  derived from `kx_shared_secrets`) in the SURB keystore, keyed by the SURB ID.

The payload encryption keys corresponding to a SURB ID may never get used, for example if the SURB
or the reply packet are lost. It is expected that nodes will only keep a limited number of keys,
for the most recently generated SURBs.

### SURB use

Constructing a reply packet from a SURB is straightforward:

- Split `header` into `kx_public`, `mac`, and `actions`.
- Write zeroes to the last 16 bytes of `payload`. Write the unencrypted [message
  fragment](./message-fragmentation.md#fragment-structure) to the other bytes.
- Derive a payload encryption key from `shared_secret` in the same way that
  `payload_encryption_key` is derived from `kx_shared_secret` [above](#secret-derivation).
- _Decrypt_ `payload` using the derived key.

The constructed packet should be sent to the mixnode with index `first_mixnode_index`. The session
is implicit: it should match the session used by the request message containing the SURB.
