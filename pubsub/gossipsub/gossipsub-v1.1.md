# gossipsub v1.1: gossipsub extensions to improve bootstrapping and attack resistance

| Lifecycle Stage | Maturity       | Status | Latest Revision |
|-----------------|----------------|--------|-----------------|
| 1A              | Draft          | Active | r1, 2020-03-20  |


Authors: [@vyzo]

Interest Group: [@yusefnapora], [@raulk], [@whyrusleeping], [@Stebalien], [@daviddias]

[@whyrusleeping]: https://github.com/whyrusleeping
[@yusefnapora]: https://github.com/yusefnapora
[@raulk]: https://github.com/raulk
[@vyzo]: https://github.com/vyzo
[@Stebalien]: https://github.com/Stebalien
[@daviddias]: https://github.com/daviddias

See the [lifecycle document][lifecycle-spec] for context about maturity level
and spec status.

[lifecycle-spec]: https://github.com/libp2p/specs/blob/master/00-framework-01-spec-lifecycle.md

---

## Overview

This document specifies extensions to [gossipsub v1.0](gossipsub-v1.0.md) intended to improve
bootstrapping and protocol attack resistant. The extensions change the algorithms that
prescribe local peer behaviour and are fully backwards compatible with v1.0 of the protocol.
Peers that implement these extensions, advertise v1.1 of the protocol using `/meshsub/1.1.0`
as the protocol string.

## Peer Exchange

Gossipsub relies on ambient peer discovery in order to find peers within a topic of interest.
This puts pressure to the implementation of a scalable peer discovery service that
can support the protocol. With Peer Exchange, the protocol can now bootstrap from a small
set of nodes, without relying on an external peer discovery service.

Peer Exchange (PX) kicks in when pruning a mesh because of oversubscription. Instead of simply
telling the pruned peer to go away, the pruning peer _may_ provide a set of other peers where the
pruned peer can connect to reform its mesh. In addition, both the pruned and the pruning peer
add a backoff period from each other, within which they will not try to regraft. Both the pruning
and the pruned peer will immediate prune a `GRAFT` within the backoff period.
The recommended duration for the back period is 1 minute.

In order to implement PX, we extend the `PRUNE` control message to include an optional set of
peers the pruned peer can connect to. This set of includes the Peer ID and a [_signed_ peer
record](https://github.com/libp2p/specs/pull/217) for each peer exchanged.
In order to facilitate transion to the usage of signed peer records within the libp2p ecosystem,
the emitting peer is allowed to omit the signed peer record if it doesn't have one.
In this case, the pruned peer will have to utilize an external service to discover addresses for
the peer, eg the DHT.

### Protobuf

The `ControlPrune` message is extended with a `peer` field as follows.

```protobuf
message ControlPrune {
	optional string topicID = 1;
	repeated PeerInfo peers = 2; // gossipsub v1.1 PX
}

message PeerInfo {
	optional bytes peerID = 1;
	optional bytes signedPeerRecord = 2;
}

```

## Flood Publishing

In gossipsub v1.0 a freshly published message is propagated through the mesh or the fanout map
if the publisher is not subscribed to the topic. In gossipsub v1.1 publishing is (optionally)
done by publishing the message to all connected peers with a score above a publish threshold
(see Peer Scoring below).

This behaviour is prescribed to counter eclipse attacks and ensure that a newly published message
from a honest node will reach all connected honest nodes and get out to the network at large.
When flood publishing is in use there is no point in utilizing a fanout map or emitting gossip when
the peer is a pure publisher not subscribed in the topic.

This behaviour also reduces message propagation latency as the message is injected to more points
in the network.

## Adaptive Gossip Dissemination

In gossipsub v1.0 gossip is emitted to a fixed number of peers, as specified by the `D_lazy`
parameter. In gossipsub v1.1 the disemmination of gossip is adaptive; instead of emitting gossip
to a fixed number of peers, we emit gossip to a percentage of our peers with a minimum of `D_lazy`
peers.

The parameter controlling the emission of gossip is called the gossip _factor_. When a node wants
to emit gossip during the heartbeat, first it selects all peers with a peer score above a gossip
threshold (see Peer Scoring below). From these peers, it randomly selects gossip factor peers with
a minimum of `D_lazy`, and emits gossip to the selected peers.

The recommended value for the gossip factor is `0.25`, which with the default of 3 rounds of gossip
per message ensures that each peer has at least 50% chance of receiving gossip about a message.
More specifically, for 3 rounds of gossip, the probability of a peer _not_ receiving gossip about
a fresh message is `(3/4)³=27/64=0.421875`. So each peer receives gossip about a fresh message with
a `0.578125` probability.

This behaviour is prescribed to counter sybil attacks and ensures that a message from a honest
node propagates in the network with high probability.

## Peer Scoring

In gossipsub v1.1 we introduce a peer scoring component: each individual peer maintains a score
for other peers. The score is locally computed by each individual peer based on observed behaviour
and is not shared. The score is a real value, computed as weighted mix of parameters,
with pluggable application specific scoring. The score is computed across all (configured) topics
with a weighted mix, such that faulty behaviour in one topic percolates to other topics.
Furthermore, the score is retained for some period of time when a peer disconnects, so that malicious
peers cannot easily reset their score when it drops to negative and well behaving
peers don't lose their status because of a disconnection.

The intention is to detect malicious or faulty behaviour and penalize the misbehaving peers
with a negative score.

### Score Thresholds

The score is plugged into various gossipsub algorithms such that peers with negative scores are
removed from the mesh. Peers with heavily negative score are further penalized or even ignored
if the score drops too low.

More specifically, the following thresholds apply:
- `0`: the baseline threshold; peers with a score below this threshold are pruned from the mesh
  during the heartbeat and ignored when looking for peers to graft. Furthermore, no PX information
  is emitted towards those peers and PX is ignored from them.
- `gossipThreshold`: when a peer's score drops below this threshold, no gossip is emitted towards
  that peer and gossip from that peer is ignored. This threshold should be negative, such that
  some information can be propagated to/from mildly negatively scoring peers.
- `publishThreshold`: when a peer's score drops below this threshold, self published messages are
  not propagated towards this peer when (flood) publishing. This threshold should be negative, and
  less than or equal to the gossip threshold.
- `graylistThreshold`: when a peer's score drops below this threshold, the peer is graylisted and
  its RPCs are ignored. This threshold must be negative, and less than the gossip/publish threshold.



## Spam Protection