# Blockchain runtime interface

The blockchain runtime provides [functions](#runtime-functions) for nodes to query the current
session status and the mixnodes for the previous/current sessions.

For the runtime to be able to register a node as a mixnode, the node must implement [a
function](#host-functions) to provide access to the node's per-session X25519 public keys.

## Runtime functions

    struct SessionStatus {
        current_index: u32,
        phase: u8,
    }

    fn MixnetApi_session_status() -> SessionStatus

`MixnetApi_session_status` returns the index and phase of the current session.

    struct Mixnode {
        kx_public: [u8; 32],
        peer_id: Vec<u8>,
        external_addresses: Vec<Vec<u8>>,
    }

    enum MixnodesErr {
        InsufficientRegistrations { num: u32, min: u32 },
        Discarded,
    }

    fn MixnetApi_prev_mixnodes() -> Result<Vec<Mixnode>, MixnodesErr>
    fn MixnetApi_current_mixnodes() -> Result<Vec<Mixnode>, MixnodesErr>

`MixnetApi_prev_mixnodes` returns the mixnodes for the previous session.
`MixnetApi_current_mixnodes` returns the mixnodes for the current session. The order of the
returned mixnodes is important; [routing actions](./sphinx.md#routing-actions) identify mixnodes by
their index in these vectors.

The following data is returned for each mixnode:

- `kx_public`; the X25519 public key for the mixnode in the session.
- `peer_id`; the libp2p peer ID of the mixnode. This is a SCALE encoded `Vec<u8>` containing a
  multihash.
- `external_addresses`; external addresses for the mixnode. Each external address is a SCALE
  encoded `Vec<u8>` containing a UTF-8 multiaddr.

Note that:

- Mixnode peer IDs and external addresses are effectively SCALE encoded twice, as runtime function
  return values are SCALE encoded. The quirky encoding of these fields matches the encoding used by
  the `ext_offchain_network_state_version_1` host function.
- The peer ID and external addresses for a mixnode are not validated _at all_ by the blockchain
  runtime. They may not even be decodable. Nodes are expected to handle this gracefully. In
  particular, failure to decode the peer ID or external addresses for one mixnode should not affect
  a node's ability to connect and send packets to other mixnodes.

These functions can return the following errors:

- `InsufficientRegistrations`; too few mixnodes were registered for the session (`num`, less than
  the minimum `min`). The mixnet is not operational in such sessions, although nodes should still
  handle traffic for the previous session in the first 3 phases.
- `Discarded`; the mixnodes for the session were discarded by the runtime. This should only be
  returned by `MixnetApi_prev_mixnodes` during the last phase of a session. The mixnodes for the
  previous session should not be needed during this phase.

## Host functions

    fn ext_mixnet_kx_public_for_session_version_1(session_index: u32) -> Option<[u8; 32]>

This should return the X25519 public key for the local node in the given session, or `None` if the
key pair was discarded already.
