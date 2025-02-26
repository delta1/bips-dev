+++
title = "Bits Denomination"
date = 2017-12-12
weight = 176
in_search_index = true

[taxonomies]
authors = ["Jimmy Song"]
status = ["Draft"]

[extra]
bip = 176
status = ["Draft"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0176.mediawiki"
+++

``` 
  BIP: 176
  Title: Bits Denomination
  Author: Jimmy Song <jaejoon@gmail.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0176
  Status: Draft
  Type: Informational
  Created: 2017-12-12
  License: BSD-2-Clause
```

## Abstract

Bits is presented here as the standard term for 100 (one hundred)
satoshis or 1/1,000,000 (one one-millionth) of a bitcoin.

## Motivation

The bitcoin price has grown over the years and once the price is past
$10,000 USD or so, bitcoin amounts under $10 USD start having enough
decimal places that it's difficult to tell whether the user is off by a
factor of 10 or not. Switching the denomination to "bits" makes
comprehension easier. For example, when BTC is $15,000 USD, $10.05 is a
somewhat confusing 0.00067 BTC, versus 670 bits, which is a lot clearer.

Additonally, reverse comparisons are easier as 59 bits being $1 is
easier to comprehend for most people than 0.000059 BTC being $1. Similar
comparisons can be made to other currencies: 1 yen being 0.8 bits, 1 won
being 0.07 bits and so on.

Potential benefits of utilizing "bits" include:

1.  Reduce user error on small bitcoin amounts.
2.  Reduce unit bias for users that want a "whole" bitcoin.
3.  Allow easier comparisons of prices for most users.
4.  Allow easier bi-directional comparisons to fiat currencies.
5.  Allows all UTXO amounts to need at most 2 decimal places, which can
    be easier to handle.

## Specification

Definition: 1 bit = 100 satoshis. Plural of "bit" is "bits." The terms
"bit" and "bits" are not proper nouns and thus should not be capitalized
unless used at the start of a sentence, etc.

All bitcoin-denominated items are encouraged to also show the
denomination in bits, either as the default or as an option.

## Rationale

As bitcoin grows in price versus fiat currencies, it's important to give
users the ability to quickly and accurately calculate prices for
transactions, savings and other economic activities. "Bits" have been
used as a denomination within the Bitcoin ecosystem for some time. The
idea of this BIP is to formalize this name. Additionally, "bits" is
likely the only other denomination that will be needed for Bitcoin as
0.01 bit = 1 satoshi, meaning that two decimal places will be sufficient
to describe any current utxo.

Existing terms used in bitcoin such as satoshi, milli-bitcoin (mBTC) and
bitcoin (BTC) do not conflict as they operate at different orders of
magnitude.

The term micro-bitcoin (µBTC) can continue to exist in tandem with the
term "bits."

## Backwards Compatibility

Software such as the Bitcoin Core GUI currently use the µBTC
denomination and can continue to do so. There is no obligation to switch
to "bits."

The term "bit" has many different definitions, but the ones of
particular note are these:

  - 1 bit = 1/8 dollar (e.g., that candy cost me 2 bits {or 1/4 dollar})
  - bit meaning some amount of data (e.g., the first bit of the version
    field is 0)
  - bit meaning strength of a cryptographic algorithm (e.g., 256-bit
    ECDSA is used in Bitcoin)

The first is a bit dated and isn't likely to confuse people dealing with
Bitcoin. The second and third are computer science terms and context
should be sufficient to figure out what the user of the word means.

## Copyright

This BIP is licensed under the BSD 2-clause license.

## Credit

It's hard to ascertain exactly who invented the term "bits," but the
term has been around for a while and the author of this BIP does not
take any credit for inventing the term.
