# Topology

The mixnodes for a session should be fully connected. That is, all mixnodes should attempt to
maintain connections to all other mixnodes. The connections should be maintained for the whole
session, as well as for the first 3 phases of the following session.

Mixnodes should also accept connections from non-mixnodes. Non-mixnodes should attempt to connect
to a small number of "gateway" mixnodes in each active session.

// TODO for implementors have an idea of small (current number in substrate against size of the mixnet for instance).
// TODO maybe state explicitely that those node are return by the blockchain runtime api, and somehow indicate that 
using validator is current runtime strategy (redundant with blockchain runtime interface part, but the part is later
in the reading).
// TODO I don t remember if opt in through the runtime, but if so, should be stated too.

## Route generation

Routes through the mixnet are always generated in the context of a session, for a single
packet/[SURB](./sphinx.md#surbs). If multiple packets are to be sent, or multiple SURBs are to be
built, a separate route should be generated for each one.

There are three kinds of route that a node may need to generate:

- From the node to a mixnode. For requests and drop cover traffic.
- From a mixnode to the node. For SURBs (sent in requests to enable replies).
- From the node to itself. For loop cover traffic.

// TODO here make it explicit if "non-mixnode" are "node".
// if so also it implies there is no mixnode to mixnode or cover between mixnode (this should maybe
be stated explicitely if it is the case).
// I don t remember too well the latest decision on it, but would be simplier to only
// have nodes (being both mixnode and non-mixnode). Actually would make sense if it is as describe
// but really the reader should see that the network good property requires external node sending
// many cover/requests.

Routes should be no longer than 7 nodes, including the source and destination nodes. The
intermediate mixnodes should be uniformly randomly chosen, subject to the following constraints:

- If the generating node is not a mixnode, any nodes immediately preceding/following it must be
  connected gateway mixnodes.
- No mixnode should appear in a route more than once, unless this is unavoidable. For example, when
  a non-mixnode is generating a loop route, if there is only a single connected gateway mixnode,
  the gateway must necessarily appear twice.
  // TODO in this case I would find it better to return an error on route generation and put back
  // the mixnet in a synching state.
  // and state that this rule cannot be check by relaying node (only when creating route).
- No node should ever appear twice consecutively in a route.
  // TODO maybe state that this rule can be enforced by relaying node by simply dropping the packet

Note that although very short routes are possible, longer routes provide more anonymity. As such,
it is recommended that by default nodes generate the longest routes possible (7 nodes).
// TODO link to corresponding sphinx packet size in sphinx.md
