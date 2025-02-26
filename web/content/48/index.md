+++
title = "Multi-Script Hierarchy for Multi-Sig Wallets"
date = 2020-12-16
weight = 48
in_search_index = true

[taxonomies]
authors = ["Fontaine"]
status = ["Proposed"]

[extra]
bip = 48
status = ["Proposed"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0048.mediawiki"
+++

``` 
  BIP: 48
  Layer: Applications
  Title: Multi-Script Hierarchy for Multi-Sig Wallets
  Author: Fontaine <dentondevelopment@protonmail.com>
  Comments-Summary: No comments
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0048
  Status: Proposed
  Type: Standards Track
  Created: 2020-12-16
  License: MIT
```

## Abstract

This BIP defines a logical hierarchy for deterministic multi-sig wallets
based on an algorithm described in BIP-0067 (BIP67 from now on),
BIP-0032 (BIP32 from now on), purpose scheme described in BIP-0043
(BIP43 from now on), and multi-account hierarchy described in BIP-0044
(BIP44 from now on).

This BIP is a particular application of BIP43.

## Copyright

This BIP falls under the MIT License.

## Motivation

The motivation of this BIP is to define the existing industry wide
practice of utilizing m/48' derivation paths in hierarchical
deterministic multi-sig wallets so that other developers may benefit
from a standard. This BIP allows for future script types to easily be
appended to the specification so that a new BIP is not required for
every future script type.

The hierarchy proposed in this paper is quite comprehensive. It allows
the handling of multiple accounts, external and internal chains per
account, multiple script types and millions of addresses per chain.

This paper was inspired from BIP44.

## Backwards compatibility

Currently a number of wallets utilize the ‎`m/48'` derivation scheme for
HD multi-sig accounts. This BIP is intended to maintain the \*existing\*
real world use of the ‎`m/48'` derivation. No breaking changes are made
so as to avoid "loss of funds" to existing users. Wallets which
currently support the ‎`m/48'` derivation will not need to make any
changes to comply with this BIP.

## Specification

### Key sorting

Any wallet that supports BIP48 inherently supports deterministic key
sorting as per BIP67 so that all possible multi-signature
addresses/scripts are derived from deterministically sorted public keys.

### Path levels

We define the following 6 levels in BIP32 path:

    m / purpose' / coin_type' / account' / script_type' / change / address_index

`h` or `'` in the path indicates that BIP32 hardened derivation is used.

Each level has a special meaning, described in the chapters below.

### Purpose

Purpose is a constant set to 48' following the BIP43 recommendation. It
indicates that the subtree of this node is used according to this
specification.

Hardened derivation is used at this level.

### Coin type

One master node (seed) can be used for multiple Bitcoin networks.
Sharing the same space for various networks has some disadvantages.

Avoiding reusing addresses across networks and improving privacy issues.

Coin type `0` for mainnet and `1` for testnet.

Hardened derivation is used at this level.

### Account

This level splits the key space into independent user identities,
following the BIP44 pattern, so the wallet never mixes the coins across
different accounts.

Users can use these accounts to organize the funds in the same fashion
as bank accounts; for donation purposes (where all addresses are
considered public), for saving purposes, for common expenses etc.

Accounts are numbered from index 0 in sequentially increasing manner.
This number is used as child index in BIP32 derivation.

Hardened derivation is used at this level.

### Script

This level splits the key space into two separate `script_type`(s). To
provide forward compatibility for future script types this specification
can be easily extended.

Currently the only script types covered by this BIP are Native Segwit
(p2wsh) and Nested Segwit (p2sh-p2wsh).

The following path represents Nested Segwit (p2sh-p2wsh) mainnet,
account 0: `1'`: Nested Segwit (p2sh-p2wsh) `m/48'/0'/0'/1'`</br>

The following path represents Native Segwit (p2wsh) mainnet, account 0:
`2'`: Native Segwit (p2wsh) `m/48'/0'/0'/2'`</br>

The recommended default for wallets is pay to witness script hash
`m/48'/0'/0'/2'`.

To add new script types submit a PR to this specification and include it
in the list above: `X'`: Future script type `m/48'/0'/0'/X'`</br>

### Change

Constant 0 is used for external chain and constant 1 for internal chain
(also known as change addresses). External chain is used for addresses
that are meant to be visible outside of the wallet (e.g. for receiving
payments). Internal chain is used for addresses which are not meant to
be visible outside of the wallet and is used for return transaction
change.

Public derivation is used at this level.

### Index

Addresses are numbered from index 0 in sequentially increasing manner.
This number is used as child index in BIP32 derivation.

Public derivation is used at this level.

## Examples

|         |         |            |          |         |                                |
| ------- | ------- | ---------- | -------- | ------- | ------------------------------ |
| network | account | script     | chain    | address | path                           |
| mainnet | first   | p2wsh      | external | first   | m / 48' / 0' / 0' / 2' / 0 / 0 |
| mainnet | first   | p2wsh      | external | second  | m / 48' / 0' / 0' / 2' / 0 / 1 |
| mainnet | first   | p2wsh      | change   | first   | m / 48' / 0' / 0' / 2' / 1 / 0 |
| mainnet | first   | p2wsh      | change   | second  | m / 48' / 0' / 0' / 2' / 1 / 1 |
| mainnet | second  | p2wsh      | external | first   | m / 48' / 0' / 1' / 2' / 0 / 0 |
| mainnet | second  | p2wsh      | external | second  | m / 48' / 0' / 1' / 2' / 0 / 1 |
| testnet | first   | p2sh-p2wsh | external | first   | m / 48' / 1' / 0' / 1' / 0 / 0 |
| testnet | first   | p2wsh      | external | second  | m / 48' / 1' / 0' / 2' / 0 / 1 |
| testnet | first   | p2wsh      | change   | first   | m / 48' / 1' / 0' / 2' / 1 / 0 |
| testnet | first   | p2wsh      | change   | second  | m / 48' / 1' / 0' / 2' / 1 / 1 |
| testnet | second  | p2wsh      | external | first   | m / 48' / 1' / 1' / 2' / 0 / 0 |
| testnet | second  | p2wsh      | external | second  | m / 48' / 1' / 1' / 2' / 0 / 1 |
| testnet | second  | p2wsh      | change   | first   | m / 48' / 1' / 1' / 2' / 1 / 0 |
| testnet | second  | p2wsh      | change   | second  | m / 48' / 1' / 1' / 2' / 1 / 1 |

## Reference

  - [BIP67 - Deterministic Pay-to-script-hash multi-signature addresses
    through public key sorting](bip-0067.mediawiki "wikilink")
  - [BIP32 - Hierarchical Deterministic
    Wallets](bip-0032.mediawiki "wikilink")
  - [BIP43 - Purpose Field for Deterministic
    Wallets](bip-0043.mediawiki "wikilink")
  - [BIP44 - Multi-Account Hierarchy for Deterministic
    Wallets](bip-0044.mediawiki "wikilink")
