+++
title = "Transaction announcements reconciliation"
date = 2019-09-25
weight = 330
in_search_index = true

[taxonomies]
authors = ["Gleb Naumenko", "Pieter Wuille"]
status = ["Draft"]

[extra]
bip = 330
status = ["Draft"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0330.mediawiki"
+++

``` 
  BIP: 330
  Layer: Peer Services
  Title: Transaction announcements reconciliation
  Author: Gleb Naumenko <naumenko.gs@gmail.com>
          Pieter Wuille <pieter.wuille@gmail.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0330
  Status: Draft
  Type: Standards Track
  Created: 2019-09-25
  License: CC0-1.0
  License-Code: MIT
```

## Abstract

This document specifies a P2P protocol extension for reconciliation of
transaction announcements <b>between 2 nodes</b>, which is a building
block for efficient transaction relay protocols (e.g.,
[Erlay](https://arxiv.org/pdf/1905.10518.pdf)). This is a step towards
increasing the connectivity of the network for almost no bandwidth cost.

## Motivation

Currently in the Bitcoin network, every 32-byte transaction ID is
announced in at least one direction between every pair of connected
peers, via INV messages. This results in high cost of announcing
transactions: *O(nodes \* connections\_per\_node)*.

A <b>reconciliation-based protocol</b> which uses the technique
suggested in this document can have better scaling properties than
INV-based flooding.

Increasing the connectivity of the network makes the network more robust
to partitioning attacks; thus, improving the bandwidth scaling of
transaction relay to *O(nodes)* (and without a high constant overhead)
would allow us to improve the security of the network by increasing
connectivity. It would also reduce the bandwidth required to run a
Bitcoin node and potentially enable more users to run full nodes.

### Erlay

[Erlay](https://arxiv.org/pdf/1905.10518.pdf) is an example of a
high-level transaction relay protocol which employs set reconciliation
for bandwidth efficiency.

Note that what we are going to describe here is a modified version from
the protocol (it is different from what is presented in the paper).

Erlay uses both flooding (announcing using INV messages to all peers)
and reconciliation to announce transactions. Flooding is expensive, so
Erlay seeks to use it only when necessary to facilitate rapid relay over
a small subset of connections.

Efficient set reconciliation is meant to deliver transactions to those
nodes which didn't receive a transaction via flooding, and also just
make sure remaining connections are in sync (directly connected pairs of
nodes are aware they have nothing to learn from each other).

Efficient set reconciliation works as follows: 1) every node keeps a
reconciliation set for each peer, in which transactions are placed which
would have been announced using INV messages absent this protocol 2)
once in a while every node chooses a peer from its reconciliation queue
to reconcile with, resulting in both sides learning the transactions
known to the other side 3) after every reconciliation round, the
corresponding reconciliation set is cleared

A more detailed description of a set reconciliation round can be found
below.

Erlay allows us to:

  - save a significant portion of the bandwidth consumed by a node
  - increase network connectivity for almost no bandwidth or latency
    cost
  - keep transaction propagation latency at the same level

This document proposes a P2P-layer extension which is required to enable
efficient reconciliation-based protocols (like Erlay) for transaction
relay.

## Specification

### New data structures

Several new data structures are introduced to the P2P protocol first, to
aid with efficient transaction relay.

#### 32-bit short transaction IDs

\= Short IDs are computed as follows:

  - Let *salt<sub>1</sub>* and *salt<sub>2</sub>* be the entropy
    contributed by both sides; see the "sendtxrcncl" message further for
    details how they are exchanged.
  - Sort the two salts such that *salt<sub>1</sub> ≤ salt<sub>2</sub>*
    (which side sent what doesn't matter).
  - Compute *h = TaggedHash("Tx Relay Salting", salt<sub>1</sub>,
    salt<sub>2</sub>)*, where the two salts are encoded in 64-bit
    little-endian byte order, and TaggedHash is specified by
    [BIP-340](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki).
  - Let *k<sub>0</sub>* be the 64-bit integer obtained by interpreting
    the first 8 bytes of *h* in little-endian byte order.
  - Let *k<sub>1</sub>* be the 64-bit integer obtained by interpreting
    the second 8 bytes of *h* in little-endian byte order.
  - Let *s = SipHash-2-4((k<sub>0</sub>,k<sub>1</sub>),wtxid)*, where
    *wtxid* is the transaction hash including witness data as defined by
    BIP141.
  - The short ID is equal to *1 + (s mod 0xFFFFFFFF)*.

This results in approximately uniformly distributed IDs in the range
*\[1..0xFFFFFFFF\]*, which is a requirement for using them as elements
in 32-bit sketches. See the next paragraph for details.

#### Short transaction ID sketches

Reconciliation-based relay uses
[PinSketch](https://www.cs.bu.edu/~reyzin/code/fuzzy.html) BCH-based
secure sketches as introduced by the [Fuzzy Extractors
paper](https://www.cs.bu.edu/~reyzin/fuzzy.html). They are a form of set
checksums with the following properties:

  - Sketches have a predetermined capacity, and when the number of
    elements in the set does not exceed the capacity, it is always
    possible to recover the entire set from the sketch by decoding the
    sketch. A sketch of nonzero b-bit elements with capacity c can be
    stored in bc bits.
  - A sketch of the [symmetric
    difference](https://en.wikipedia.org/wiki/Symmetric_difference)
    between the two sets (i.e., all elements that occur in one but not
    both input sets), can be obtained by combining the sketches of those
    sets.

The sketches used here consists of elements of the [finite
field](https://en.wikipedia.org/wiki/Finite_field) *GF(2<sup>32</sup>)*.
Specifically, we represent finite field elements as polynomials in *x*
over *GF(2)* modulo *x<sup>32</sup>7</sup> + x<sup>3</sup> +
x<sup>2</sup> + 1*. To map integers to finite field elements, simply
treat each bit *i* (with value *2<sup>i</sup>*) in the integer as the
coefficient of *x<sup>i</sup>* in the polynomial representation. For
example the integer *101 = 2<sup>6</sup> + 2<sup>5</sup> + 2<sup>2</sup>
+ 1* is mapped to field element *x<sup>6</sup> + x<sup>5</sup> +
x<sup>2</sup> + 1*. These field elements can be added and multiplied
together, but the specifics of that are out of scope for this document.

A short ID sketch with capacity *c* consists of a sequence of *c* field
elements. The first is the sum of all short IDs in the set, the second
is the sum of the 3rd powers of all short IDs, the third is the sum of
the 5th powers etc., up to the last element with is the sum of the
*(2c-1)*th powers. These elements are then encoded as 32-bit integers in
little endian byte order, resulting in a *4c*-byte serialization.

The following Python 3.2+ code implements the creation of sketches:

    FIELD_BITS = 32
    FIELD_MODULUS = (1 << FIELD_BITS) + 0b10001101
    
    def mul2(x):
        """Compute 2*x in GF(2^FIELD_BITS)"""
        return (x << 1) ^ (FIELD_MODULUS if x.bit_length() >= FIELD_BITS else 0)
    
    def mul(x, y):
        """Compute x*y in GF(2^FIELD_BITS)"""
        ret = 0
        for bit in [(x >> i) & 1 for i in range(x.bit_length())]:
            ret, y = ret ^ bit * y, mul2(y)
        return ret
    
    def create_sketch(shortids, capacity):
        """Compute the bytes of a sketch for given shortids and given capacity."""
        odd_sums = [0 for _ in range(capacity)]
        for shortid in shortids:
            squared = mul(shortid, shortid)
            for i in range(capacity):
                odd_sums[i] ^= shortid
                shortid = mul(shortid, squared)
        return b''.join(elem.to_bytes(4, 'little') for elem in odd_sums)

The [minisketch](https://github.com/sipa/minisketch/) library implements
the construction, merging, and decoding of these sketches efficiently.

### Intended Protocol Flow

Set reconciliation primarily consists of the transmission and decoding
of a reconciliation set sketch upon request.

Since sketches are based on the WTXIDs, the negotiation and support of
Erlay should be enabled only if both peers signal
[BIP-339](https://github.com/bitcoin/bips/blob/master/bip-0339.mediawiki)
support.

[framed|center|Protocol
flow](File:bip-0330/recon_scheme_merged.png "wikilink")

#### Sketch extension

If a node is unable to reconstruct the set difference from the received
sketch, the node then makes a request for sketch extension. The peer
would then send an extension, which is a sketch of a higher capacity
(allowing to decode more differences) over the same transactions minus
the sketch part which was already sent initially (to save bandwidth). To
allow this optimization, the initiator is supposed to locally store a
sketch received initially. This optimization is possible because
extending a sketch is just concatenating new elements to an array.

### New messages

Several new protocol messages are added: sendtxrcncl, reqrecon, sketch,
reqsketchext, reconcildiff. This section describes their serialization,
contents, and semantics.

In what follows, all integers are serialized in little-endian byte
order. Boolean values are encoded as a single byte that must be 0 or 1
exactly. Arrays are serialized with the CompactSize prefix that encodes
their length, as is common in other P2P messages.

#### sendtxrcncl

The sendtxrcncl message announces support for the reconciliation
protocol. It is expected to be only sent once, and ignored by nodes that
don't support it.

Should be sent before "verack" and accompanied by "wtxidrelay" (in any
order).

If "sendtxrcncl" was sent after "verack", the sender should be
disconnected.

If "sendtxrcncl" was sent before "verack", but by "verack" the
"wtxidrelay" message was not received, "sendtxrcncl" should be ignored.
The connection should proceed normally, but as if reconciliation was not
supported.

Must not be sent if peer specified no support for transaction relay
(fRelay=0) in "version". Otherwise, the sender should be disconnected.

Its payload consists of:

| Data type | Name    | Description                                                                                                                                                          |
| --------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| uint32    | version | Sender must set this to 1 currently, otherwise receiver should ignore the message. v1 is the lowest protocol version, everything below that is a protocol violation. |
| uint64    | salt    | The salt used in the short transaction ID computation.                                                                                                               |

After both peers have confirmed support by sending "sendtxrcncl", the
initiator of the P2P connection assumes the role of reconciliation
initiator (will send "reqrecon" messages) and the other peer assumes the
role of reconciliation responder (will respond to "reqrecon" messages).
"reqrecon" messages can only be sent by the reconciliation initiator.

#### reqrecon

The reqrecon message initiates a reconciliation round.

| Data type | Name      | Description                                                                                                                                            |
| --------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| uint16    | set\_size | Size of the sender's reconciliation set, used to estimate set difference.                                                                              |
| uint16    | q         | Coefficient used to estimate set difference. Multiplied by PRECISION=(2^15) - 1 and rounded up by the sender and divided by PRECISION by the receiver. |

Upon receipt of a "reqrecon" message, the receiver:

  - Constructs and sends a "sketch" message (see below), with a sketch
    of certain *capacity=f(set\_size, local\_set\_size, q)* (the exact
    function is suggested below), where *local\_set\_size* represents
    size of the receiver's reconciliation set.
  - Makes a snapshot of their current reconciliation set, and clears the
    set itself. The snapshot is kept until a "reconcildiff" message is
    received by the node.

No new "reqrecon" message can be sent until a "reconcildiff" message is
sent.

#### sketch

The sketch message is used to communicate a sketch required to perform
set reconciliation.

| Data type | Name   | Description                                        |
| --------- | ------ | -------------------------------------------------- |
| byte\[\]  | skdata | The sketch of the sender's reconciliation snapshot |

The sketch message may be received in two cases.

1\. Initial sketch. Upon receipt of a "sketch" message, a node computes
the difference sketch by combining the received sketch with a sketch
computed locally for a corresponding reconciliation set. The receiving
node then tries to decode the difference sketch and based on the result:

  - If the decoding failed, the receiving node requests an extension
    sketch by sending a "reqsketchext" message. Alternatively, the node
    may terminate the reconciliation right away by sending a
    "reconcildiff" message is sent with the failure flag set
    (success=false).
  - If the decoding succeeded, a "reconcildiff" message with
    success=true.

The receiver also makes snapshot of their current reconciliation set,
and clears the set itself. The snapshot is kept until a "reconcildiff"
message is sent by the node. It is needed to enable sketch extension.

2\. Sketch extension. By combining the sketch extension with the
initially received sketch, an extended sketch is obtained. The receiving
node then computes the extended difference sketch by combining the
received extended sketch with an extended sketch computed locally over a
corresponding reconciliation set snapshot. The receiving node then tries
to decode the extended difference sketch and based on the result:

  - If the decoding failed, the receiving node terminates the
    reconciliation right away by sending a "reconcildiff" message is
    sent with the failure flag set (success=false).
  - If the decoding succeeded, a "reconcildiff" message with
    success=true.

In either cases, a "reconcildiff" with success=false should also be
accompanied with announcing all transactions from the reconciliation set
(or set snapshot if failed after extension) as a fallback to flooding. A
"reconcildiff" with success=true should contain unknown short IDs of the
transactions from the decoded difference, corresponding to the
transactions missing on the sender's side. Known short IDs from the
difference correspond to what the receiver of the message is missing,
and they should be announced via an "inv" message.

#### reqsketchext

The reqsketchext message is used by reconciliation initiator to signal
that initial set reconciliation has failed and a sketch extension is
needed to find set difference.

It has an empty payload.

Upon receipt of a "reqsketchext" message, a node responds to it with a
"sketch" message, which contains a sketch extension: a sketch (of the
same transactions sketched initially) of higher capacity without the
part sent initially.

#### reconcildiff

The reconcildiff message is used by reconciliation initiator to announce
transactions which are found to be missing during set reconciliation on
the sender's side.

| Data type  | Name          | Description                                                                   |
| ---------- | ------------- | ----------------------------------------------------------------------------- |
| uint8      | success       | Indicates whether sender of the message succeeded at set difference decoding. |
| uint32\[\] | ask\_shortids | The short IDs that the sender did not have.                                   |

Upon receipt a "reconcildiff" message with *success=1* (reconciliation
success), a node sends an "inv" message for the transactions requested
by 32-bit IDs (first vector) containing their wtxids (with parent
transactions occuring before their dependencies). If *success=0*
(reconciliation failure), receiver should announce all transactions from
the reconciliation set via an "inv" message. In both cases, transactions
the sender of the message thinks the receiver is missing are announced
via an "inv" message. The regular "inv" deduplication should apply.

The <b>snapshot</b> of the corresponding reconciliation set is cleared
by the sender and the receiver of the message.

The sender should also send their own "inv" message along with the
reconcildiff message to announce transactions which are missing on the
receiver's side.

## Local state

This BIP suggests a stateful protocol and it requires storing several
variables at every node to operate properly.

#### Reconciliation salt

When negotiating reconciliation support, peers send each other their
contribution to the reconciliation salt (see how we construct short IDs
above). These salts (or just the resulting salt) should be stored on
both sides of the connection.

#### Reconciliation sets

Every node stores a set of wtxids for every peer which supports
transaction reconciliation, representing the transactions which would
have been sent according to the regular flooding protocol. Incoming
transactions are added to sets when those transactions are received (if
they satisfy the policies such as minimum fee set by a peer). A
reconciliation set is moved to the corresponding set snapshot after the
transmission of the initial sketch.

#### Reconciliation set snapshot

After transmitting the initial sketch (either sending or receiving of
the reconcildiff message), every node should store the snapshot of the
current reconciliation set, and clear the set. This is important to make
sketch extension more stable (extension should be computed over the set
snapshot). Otherwise, extension would contain transactions received
after sending out the initial sketch. The snapshot is cleared after the
end of the reconciliation round (sending or receiving of the
reconcildiff message).

#### Sketch capacity estimation and q-coefficient

Earlier we suggested that upon receiving a reconciliation request, a
node should estimate the sketch capacity it should send:
*capacity=f(set\_size, local\_set\_size, q)*.

We suggest the following function: *capacity=|set\_size -
local\_set\_size| + q \* min(set\_size, local\_set\_size) + c*.

Intuitively, *q* represents the discrepancy in sets: the closer the sets
are, the lower optimal *q* is. Per the Erlay paper, *q* should be
derived as an optimal *q* value for the previous reconciliation with a
given peer, once the actual set sizes and set difference are known. For
example, if in previous round *set\_size=30* and *local\_set\_size=20*,
and the \*actual\* difference was *12*, then a node should compute *q*
as following: *q=(12 - |30-20|) / min(30, 20)=0.1*

The derivation of *q* can be changed according to the version of the
protocol. For example, a static value could be chosen for simplicity.
However, we suggest that *q* remains a parameter sent in every
reconciliation request to enable future compatibility with more
sophisticated (non-static) choices of this parameter.

As for the *c* parameter, it is suggested to use *c=1* to avoid sending
empty sketches and reduce the overhead caused by under-estimations.

## Backward compatibility

Older clients remain fully compatible and interoperable after this
change.

Clients which do not implement this protocol remain fully compatible
after this change using existing protocols, because transaction
announcement reconciliation is used only for peers that negotiate
support for it.

## Rationale

#### Why use PinSketch for set reconciliation?

PinSketch is more bandwidth efficient than IBLT, especially for the
small differences in sets we expect to operate over. PinSketch is as
bandwidth efficient as CPISync, but PinSketch has quadratic decoding
complexity, while CPISync have cubic decoding complexity. This makes
PinSketch significantly faster.

#### Why use 32-bit short transaction IDs?

To use Minisketch in practice, transaction IDs should be shortened
(ideally, not more than 64 bits per element). A small number of bits per
transaction also allows saving extra bandwidth and make operations over
sketches faster. According to our estimates, 32 bits provides low
collision rate in a non-adversarial model (which is enabled by using
independent salts per-link).

#### Why use sketch extensions instead of bisection?

Bisection is an alternative to sketch extensions, per which a second
sketch with the same initial capacity is computed over half of the txID
space. Due to the linearity of sketches, transmitting just this one
allows a reconciliation initiator to compute the sketch of the same
capacity of another half. Two sketches allow the initiator to
reconstruct twice as many differences as was allowed by an initial
sketch.

In practice this allows the initiator to amortize the bandwidth overhead
of initial reconciliation failure, similarly to extension sketches,
making the overhead negligible.

The main benefit of sketch extensions is a much simpler implementation.
Implementing bisection is hard (see
[implementation](https://github.com/naumenkogs/bitcoin/commit/b5c92a41e4cc0599504cf838d20212f1a403e573))
because, in the end, we have to operate with two sketches and handle
scenarios where one sketch decoded and another sketch failed.

It becomes even more difficult if in the future we decide to allow more
than one extension/bisection. Bisection in this case have to be
recursive (and spawn 4/8/16/... sketches), while for extensions we
always end up with one extended sketch.

Sketch extensions are also more flexible: extending a sketch of capacity
10 with 4 more means just computing a sketch of capacity 14 and sending
the extension, while for bisection increasing the capacity to something
different than 10\*2/10\*4/10\*8/... is sophisticated
implementation-wise.

The only advantage of bisection is that it doesn't require computing
sketches of higher capacities (exponential cost). We believe that since
the protocol is currently designed to operate in the conditions where
sketches usually have at most the capacity of 20, this efficiency is not
crucial.

## Implementation

<https://github.com/bitcoin/bitcoin/pull/21515>

## Acknowledgments

A large fraction of this proposal was done during designing Erlay with
Gregory Maxwell, Sasha Fedorova and Ivan Beschastnikh. We would like to
thank Suhas Daftuar for contributions to the design and BIP structure.
We would like to thank Ben Woosley for contributions to the high-level
description of the idea.

## Copyright

This document is licensed under the Creative Commons CC0 1.0 Universal
license.
