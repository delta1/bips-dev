+++
title = "Disable transaction relay message"
date = 2020-09-03
weight = 338
in_search_index = true

[taxonomies]
authors = ["Suhas Daftuar"]
status = ["Draft"]

[extra]
bip = 338
status = ["Draft"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0338.mediawiki"
+++

``` 
  BIP: 338
  Layer: Peer Services
  Title: Disable transaction relay message
  Author: Suhas Daftuar <sdaftuar@chaincode.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0338
  Status: Draft
  Type: Standards Track
  Created: 2020-09-03
  License: BSD-2-Clause
```

## Abstract

This BIP describes a change to the p2p protocol to allow a node to tell
a peer that a connection will not be used for transaction relay, to
support block-relay-only connections that are currently in use on the
network.

## Motivation

This proposal is part of an effort to increase the number of inbound
connections that a peer can service, by distinguishing peers which will
not relay transactions from those that do.

Since 2019, software has been deployed\[1\] which initiates connections
on the Bitcoin network and sets the transaction relay field (introduced
by BIP 37 and also defined in BIP 60) to false, to prevent transaction
relay from occurring on the connection. Additionally, addr messages
received from the peer are ignored by this software.

The purpose of these connections is two-fold: by making additional
low-bandwidth connections on which blocks can propagate, the robustness
of a node to network partitioning attacks is strengthened. Additionally,
by not relaying transactions and ignoring received addresses, the
ability of an adversary to learn the complete network graph (or a
subgraph) is reduced\[2\], which in turn increases the cost or
difficulty to an attacker seeking to carry out a network partitioning
attack (when compared with having such knowledge).

The low-bandwidth / minimal-resource nature of these connections is
currently known only by the initiator of the connection; this is because
the transaction relay field in the version message is not a permanent
setting for the lifetime of the connection. Consequently, a node
receiving an inbound connection with transaction relay disabled cannot
distinguish between a peer that will never enable transaction relay (as
described in BIP 37) and one that will. Moreover, the node also cannot
determine that the incoming connection will ignore relayed addresses;
with that knowledge a node would likely choose other peers to receive
announced addresses instead.

This proposal adds a new, optional message that a node can send a peer
when initiating a connection to that peer, to indicate that connection
should not be used for transaction relay for the connection's lifetime.
In addition, without a current mechanism to negotiate whether addresses
should be relayed on a connection, this BIP suggests that address
messages not be sent on links where transaction relay has been disabled.

After this BIP is deployed, nodes could more easily implement inbound
connection limiting that differentiates low-resource nodes (such as
those sending disabletx) from full-relay peers, potentially allowing for
an increase in the number of block-relay-only connections that can be
made on the network.

## Specification

1.  A new disabletx message is added, which is defined as an empty
    message with message type set to "disabletx".
2.  The protocol version of nodes implementing this BIP must be set to
    70017 or higher.
3.  If a node sets the transaction relay field in the version message to
    a peer to false, then the disabletx message MAY also be sent in
    response to a version message from that peer if the peer's protocol
    version is \>= 70017. If sent, the disabletx message MUST be sent
    prior to sending a verack.
4.  A node MUST NOT send the disabletx message if the transaction relay
    field in the version message is omitted or set to true.
5.  A node that has sent or received a disabletx message to/from a peer
    MUST NOT send any of these messages to the peer:
    1.  inv messages for transactions
    2.  notfound messages for transactions
    3.  getdata messages for transactions
    4.  getdata messages for merkleblock (BIP 37)
    5.  filteradd/filterload/filterclear (BIP 37)
    6.  feefilter (BIP 133)
    7.  mempool (BIP 35)
    8.  tx message
6.  It is RECOMMENDED that a node that has sent or received a disabletx
    message to/from a peer not send any of these messages to the peer:
    1.  addr/getaddr
    2.  addrv2 (BIP 155)
7.  The behavior regarding sending or processing other message types is
    not specified by this BIP.
8.  Nodes MAY decide to not remain connected to peers that send this
    message (for example, if trying to find a peer that will relay
    transactions).

## Compatibility

Nodes with protocol version \>= 70017 that do not implement this BIP,
and nodes with protocol version \< 70017, will continue to remain
compatible with implementing software: transactions would not be relayed
to peers sending the disabletx message (provided that BIP 37 or BIP 60
has been implemented), and while periodic address relay may still take
place, software implementing this BIP should not be disconnecting such
peers solely for that reason.

Disabling address relay is suggested but not required by this BIP, to
allow for future protocol extensions that might specify more carefully
how address relay is to be negotiated. This BIP's recommendations for
software to not relay addresses is intended to be interpreted as
guidance in the absence of any such future protocol extension, to
accommodate existing software behavior.

Note that all messages specified in BIP 152, including blocktxn and
getblocktxn, are permitted between peers that have sent/received a
disabletx message, subject to the feature negotiation of BIP 152.

This proposal is compatible with, but independent of, BIP 37.

## Implementation

<https://github.com/bitcoin/bitcoin/pull/20726>

## References

1.  Bitcoin Core has [implemented this
    functionality](https://github.com/bitcoin/bitcoin/pull/15759) since
    version 0.19.0.1, released in November 2019.
2.  For example, see
    <https://www.cs.umd.edu/projects/coinscope/coinscope.pdf> and
    <https://arxiv.org/pdf/1812.00942.pdf>.

## Copyright

This BIP is licensed under the 2-clause BSD license.
