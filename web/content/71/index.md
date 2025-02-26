+++
title = "Payment Protocol MIME types"
date = 2013-07-29
weight = 71
in_search_index = true

[taxonomies]
authors = ["Gavin Andresen"]
status = ["Final"]

[extra]
bip = 71
status = ["Final"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0071.mediawiki"
+++

``` 
  BIP: 71
  Layer: Applications
  Title: Payment Protocol MIME types
  Author: Gavin Andresen <gavinandresen@gmail.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0071
  Status: Final
  Type: Standards Track
  Created: 2013-07-29
```

## Abstract

This BIP defines a MIME (RFC 2046) Media Type for Bitcoin payment
request messages.

## Motivation

Wallet or server software that sends payment protocol messages over
email or http should follow Internet standards for properly
encapsulating the messages.

## Specification

The Media Type (Content-Type in HTML/email headers) for bitcoin protocol
messages shall be:

|                |                                    |
| -------------- | ---------------------------------- |
| Message        | Type/Subtype                       |
| PaymentRequest | application/bitcoin-paymentrequest |
| Payment        | application/bitcoin-payment        |
| PaymentACK     | application/bitcoin-paymentack     |

Payment protocol messages are encoded in binary.

## Example

A web server generating a PaymentRequest message to initiate the payment
protocol would precede the binary message data with the following
headers:

    Content-Type: application/bitcoin-paymentrequest
    Content-Transfer-Encoding: binary
