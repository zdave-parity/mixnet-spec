# Introduction

A Substrate Mix Network enables anonymous submission of transactions to a blockchain. The Substrate
Mix Network design is loosely based on
[Loopix](https://www.usenix.org/system/files/conference/usenixsecurity17/sec17-piotrowska.pdf).

A Substrate Mix Network has two main components:

- A network of nodes.
- A blockchain, which provides consensus on which nodes should operate as "mixnodes", and accepts
  the mixnet submitted transactions.

This specification details the behaviour required of a node. It is primarily aimed at those
creating alternate node implementations.

This specification does _not_ attempt to:

- Describe the blockchain runtime logic; only the interface to the runtime is covered.
- Describe any particular node implementation.
- Explain design decisions.
