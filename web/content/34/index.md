+++
title = "Block v2, Height in Coinbase"
date = 2012-07-06
weight = 34
in_search_index = true

[taxonomies]
authors = ["Gavin Andresen"]
status = ["Final"]

[extra]
bip = 34
status = ["Final"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0034.mediawiki"
+++

``` 
  BIP: 34
  Layer: Consensus (soft fork)
  Title: Block v2, Height in Coinbase
  Author: Gavin Andresen <gavinandresen@gmail.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0034
  Status: Final
  Type: Standards Track
  Created: 2012-07-06
```

## Abstract

Bitcoin blocks and transactions are versioned binary structures. Both
currently use version 1. This BIP introduces an upgrade path for
versioned transactions and blocks. A unique value is added to newly
produced coinbase transactions, and blocks are updated to version 2.

## Motivation

1.  Clarify and exercise the mechanism whereby the bitcoin network
    collectively consents to upgrade transaction or block binary
    structures, rules and behaviors.
2.  Enforce block and transaction uniqueness, and assist unconnected
    block validation.

## Specification

1.  Treat transactions with a version greater than 1 as non-standard
    (official Satoshi client will not mine or relay them).
2.  Add height as the first item in the coinbase transaction's
    scriptSig, and increase block version to 2. The format of the height
    is "minimally encoded serialized CScript" -- first byte is number of
    bytes in the number (will be 0x03 on main net for the next 150 or so
    years with 2<sup>23</sup>-1 blocks), following bytes are
    little-endian representation of the number (including a sign bit).
    Height is the height of the mined block in the block chain, where
    the genesis block is height zero (0).
3.  75% rule: If 750 of the last 1,000 blocks are version 2 or greater,
    reject invalid version 2 blocks. (testnet3: 51 of last 100)
4.  95% rule ("Point of no return"): If 950 of the last 1,000 blocks are
    version 2 or greater, reject all version 1 blocks. (testnet3: 75 of
    last 100)

## Backward compatibility

All older clients are compatible with this change. Users and merchants
should not be impacted. Miners are strongly recommended to upgrade to
version 2 blocks. Once 95% of the miners have upgraded to version 2, the
remainder will be orphaned if they fail to upgrade.

## Implementation

<https://github.com/bitcoin/bitcoin/pull/1526>

## Result

Block number 227,835 (timestamp 2013-03-24 15:49:13 GMT) was the last
version 1 block.
