+++
title = "Segregated Witness (second deployment)"
date = 2017-04-14
weight = 149
in_search_index = true

[taxonomies]
authors = ["Shaolin Fry"]
status = ["Withdrawn"]

[extra]
bip = 149
status = ["Withdrawn"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0149.mediawiki"
+++

``` 
  BIP: 149
  Layer: Consensus (soft fork)
  Title: Segregated Witness (second deployment)
  Author: Shaolin Fry <shaolinfry@protonmail.ch>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0149
  Status: Withdrawn
  Type: Standards Track
  Created: 2017-04-14
  License: BSD-3-Clause
           CC0-1.0
```

## Abstract

This document specifies a user activated soft fork for
[BIP141](bip-0141.mediawiki "wikilink"),
[BIP143](bip-0143.mediawiki "wikilink") and
[BIP147](bip-0147.mediawiki "wikilink") using versionbits with
guaranteed lock-in.

## Motivation

Miners have been reluctant to signal the BIP9 segwit deployment despite
a large portion of the Bitcoin ecosystem who want the soft fork
activated. This BIP specifies a user activated soft fork (UASF) that
deploys segwit again using versionbits with guaranteed lock-in on
timeout if the BIP is not already locked-in or activated by the timeout.
This ensures users have sufficient time to prepare and no longer require
a miner supermajority, while still allowing for an earlier miner
activated soft fork (MASF).

## Reference implementation

The reference implementation will refuse to run on Bitcoin mainnet
before 7 November 2017, and can only be run on testnet and regtest until
then.

<https://github.com/bitcoin/bitcoin/compare/master>...shaolinfry:uasegwit-flagday

## Specification

This deployment will set service bit (1\<\<5) as NODE\_UAWITNESS.

## Deployment

This BIP should only be deployed if BIP9-segwit fails to lock-in or
activate before timeout on 15 November 2017. This BIP cannot be deployed
before 15 November 2017.

This BIP will be deployed by BIP8 with the name "segwit" and using bit
1.

For Bitcoin mainnet, the BIP8 starttime will be midnight 16 November
2017 UTC (Epoch timestamp 1510790400) and BIP8 timeout will be 4 July
2018 UTC (Epoch timestamp 1530662400).

For Bitcoin testnet, segwit is already activated so no deployment is
specified.

## Backwards Compatibility

This deployment reuses the GBT deployment name "segwit" to maintain
compatibility with existing mining software.

This deployment is incompatible with the BIP9-segwit deployment and
should not be run concurrently with it.

## Rationale

The **starttime** of this BIP is after the BIP9-segwit timeout to remove
compatibility issues with old nodes.

## References

[Mailing list
discussion](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-April/014234.html)

[BIP8](bip-0008.mediawiki "wikilink")

[BIP9](bip-0009.mediawiki "wikilink")

[BIP141](bip-0141.mediawiki "wikilink")

[BIP143](bip-0143.mediawiki "wikilink")

[BIP147](bip-0147.mediawiki "wikilink")

## Copyright

This document is dual licensed as BSD 3-clause, and Creative Commons CC0
1.0 Universal.
