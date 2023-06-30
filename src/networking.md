# Networking

Nodes send mixnet packets to each other using a Substrate notifications protocol. The protocol name
is derived from the genesis hash of the blockchain:

    /{genesis_hash_hex}/mixnet/1

or (in the case of a blockchain fork):

    /{genesis_hash_hex}/{fork_id}/mixnet/1

Notifications with sizes not matching the [Sphinx packet size](./sphinx.md#packet-structure) should
be discarded. All other notifications should be handled as Sphinx packets.

Nodes should use the [peer IDs and external addresses published on the
blockchain](./blockchain-runtime-interface.md#runtime-functions) to connect to mixnodes according
to the [mixnet topology](./topology.md). If no external addresses have been published for a
mixnode, or none of them work, nodes should attempt to discover addresses using the libp2p DHT.

// Here I am not sure, do we have the Dht? or if we are talking about discovery dht, we should name
// it. Also it implies that we default to the common validator address (an implementor could make all mixnet
// in a different address (different node than the validating one. Even if not doable with current implementation, this sounds like easy double signing slashing for wrongly configured validators though, also may need a switch in the registration process to register on a single node, may be some smarter conf where we call the transaction to the validator node from the mixnet node to avoid duplicating key, but that is more work)).

// Second thought is this DHT resolution on the discovery risk of spamming validator nodes that purposedly did not run the mixnet, maybe it is a good thing though?

All mixnet node peer IDs should be derived from Ed25519 public keys, "hashed" with the identity
hash function. Nodes may assume this, although a bad peer ID published for one mixnode should not
interfere with the ability of a node to connect and send packets to other mixnodes.
