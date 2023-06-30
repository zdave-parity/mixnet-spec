# Sessions

The blockchain runtime divides time up into sessions, indexed starting from 0. Note that mixnet
sessions need not be the same as eg BABE sessions.

In the context of a session, the network nodes are split into two classes: mixnodes and
non-mixnodes. Mixnodes are responsible for mixing traffic. Non-mixnodes may send requests into the
mixnet and receive replies from the mixnet, but do not mix traffic. Non-mixnodes can join and leave
the mixnet freely, but it is expected that if a node is a mixnode in a session, it will participate
for the duration of the session.

The mixnodes for each session are determined by the blockchain runtime. The mixnodes for a session
need not be related in any way to the mixnodes for the previous/following sessions.

Every node generates an X25519 key pair per session. Nodes use these keys to generate shared
secrets for packet encryption and such; see the [Sphinx chapter](./sphinx.md#key-exchange) for more
details. Mixnode public keys are published on the blockchain and are thus available to all nodes.
Non-mixnode public keys are not published.

When a packet is constructed, it is constructed explicitly for a particular session. A packet's
session is implicitly determined by the X25519 keys used to build it (again, see the [Sphinx
chapter](./sphinx.md#mac-verification)).

## Phases

Mixnet traffix is switched from one session to the next gradually, not instantly. Each mixnet
session is divided into 4 phases. The current phase is determined by the blockchain runtime. The
phases are as follows:

| Phase | Previous session traffic          | Current sesssion traffic          | Default session |
|-------|-----------------------------------|-----------------------------------|-----------------|
| 0     | All, half rate                    | Cover and forward only, half rate | Previous        |
| 1     | All, half rate                    | All, half rate                    | Current         |
| 2     | Cover and forward only, half rate | All, half rate                    | Current         |
| 3     | None                              | All, full rate                    | Current         |

The session traffic columns indicate which packet classes should be sent using the previous/current
session keys/mixnodes during each phase:

- None: no packets should be sent.
- Cover and forward only: cover packets should be generated and forwarded. Request/reply packets
  should _not_ be sent.
- All: no restrictions.

In the first 3 phases (where traffic is sent using both the previous and current session
keys/mixnodes), cover and request/reply traffic should be sent at half the usual rate in each
session.

The default session column indicates which session should be preferred for new requests.

Note that once the last phase has been reached, the X25519 key pairs for the previous session are
no longer needed, and nodes should discard them.

// TODO here would be good to have the explanation that phase 0 is at the very start of a session
// and state that session duration are defined by the runtime blockchain api (redundant with runtime-interface.md but helps the reading).
// Could also indicate that implementor are not forced to follow these (I don't remember if we check rate anymore, I guess we don't).
