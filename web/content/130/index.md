+++
title = "sendheaders message"
date = 2015-05-08
weight = 130
in_search_index = true

[taxonomies]
authors = ["Suhas Daftuar"]
status = ["Proposed"]

[extra]
bip = 130
status = ["Proposed"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0130.mediawiki"
+++

``` 
  BIP: 130
  Layer: Peer Services
  Title: sendheaders message
  Author: Suhas Daftuar <sdaftuar@chaincode.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0130
  Status: Proposed
  Type: Standards Track
  Created: 2015-05-08
  License: PD
```

## Abstract

Add a new message, "sendheaders", which indicates that a node prefers to
receive new block announcements via a "headers" message rather than an
"inv".

## Motivation

Since the introduction of "headers-first" downloading of blocks in 0.10,
blocks will not be processed unless they are able to connect to a
(valid) headers chain. Consequently, block relay generally works as
follows:

1.  A node (N) announces the new tip with an "inv" message, containing
    the block hash
2.  A peer (P) responds to the "inv" with a "getheaders" message (to
    request headers up to the new tip) and a "getdata" message for the
    new tip itself
3.  N responds with a "headers" message (with the header for the new
    block along with any preceding headers unknown to P) and a "block"
    message containing the new block

However, in the case where a new block is being announced that builds on
the tip, it would be generally more efficient if the node N just
announced the block header for the new block, rather than just the block
hash, and saved the peer from generating and transmitting the getheaders
message (and the required block locator).

In the case of a reorg, where 1 or more blocks are disconnected, nodes
currently just send an "inv" for the new tip. Peers currently are able
to request the new tip immediately, but wait until the headers for the
intermediate blocks are delivered before requesting those blocks. By
announcing headers from the last fork point leading up to the new tip in
the block announcement, peers are able to request all the intermediate
blocks immediately.

## Specification

1.  The sendheaders message is defined as an empty message where
    pchCommand == "sendheaders"
2.  Upon receipt of a "sendheaders" message, the node will be permitted,
    but not required, to announce new blocks by sending the header of
    the new block (along with any other blocks that a node believes a
    peer might need in order for the block to connect).
3.  Feature discovery is enabled by checking protocol version \>= 70012

## Additional constraints

As support for sendheaders is optional, software that implements this
may also optionally impose additional constraints, such as only honoring
sendheaders messages shortly after a connection is established.

## Backward compatibility

Older clients remain fully compatible and interoperable after this
change.

## Implementation

<https://github.com/bitcoin/bitcoin/pull/6494>

## Copyright

This document is placed in the public domain.
