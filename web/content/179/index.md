+++
title = "Name for payment recipient identifiers"
date = 2019-10-17
weight = 179
in_search_index = true

[taxonomies]
authors = ["Emil Engler", "Luke Dashjr"]
status = ["Draft"]

[extra]
bip = 179
status = ["Draft"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0179.mediawiki"
+++

``` 
  BIP: 179
  Title: Name for payment recipient identifiers
  Author: Emil Engler <me@emilengler.com>
          Luke Dashjr <luke+bip@dashjr.org>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0179
  Status: Draft
  Type: Informational
  Created: 2019-10-17
  License: CC0-1.0
```

## Abstract

This BIP proposes a new term for 'address'

## Specification

The new term is: *Bitcoin* **Invoice** *Address*

The *Bitcoin* and *Address* parts are optional. The address suffix
should only be used as a transitional step.

A *Bitcoin* Invoice *Address* is a string of characters that can be used
to indicate the intended recipient and purpose of a transaction.

## Motivation

Bitcoin addresses are intended to be only used **once** and you should
generate a new one for every new incoming payment. The term 'address'
however indicates consistency because nearly everything on the internet
or the offline world with the term 'address' is something that rarely or
even never changes (postal address, email address, IP addresses (depends
heavily on the provider), etc.) The motivation for this BIP is to change
the term address to something that indicates that the address is
connected to a single transaction.

## Rationale

The reason why we use *Bitcoin Invoice Address* or just *Invoice* is to
emphasize that it is single-use. The terms *Bitcoin* and *Address* are
optional for the following reasons: For *Bitcoin*:

  - Useful for multicoin wallets to indicate that it belongs to Bitcoin
  - Indicates a difference between a lightning and an on-chain invoice

For *Address*:

  - To not confuse users with a completely new term
  - To show that it is where you send something to
  - To not break backwards compatibility

This gives us the four following possibilities:

  - Bitcoin Invoice Address
  - Bitcoin Invoice
  - Invoice Address
  - Invoice

## Backwards Compatibility

To avoid issues, the 'Address' suffix is permitted, but not recommended.
The suffix 'Address' remains so users should be immediately able to
recognize it until the new term is widely known.

## Acknowledgements

Thanks to Chris Belcher for the suggestion of the term 'Bitcoin Invoice
Address'

## Copyright

This BIP is released under CC0-1.0 and therefore Public Domain.
