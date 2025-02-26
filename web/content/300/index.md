+++
title = "Hashrate Escrows (Consensus layer)"
date = 2017-08-14
weight = 300
in_search_index = true

[taxonomies]
authors = ["Paul Sztorc", "CryptAxe"]
status = ["Draft"]

[extra]
bip = 300
status = ["Draft"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0300.mediawiki"
+++

``` 
  BIP: 300
  Layer: Consensus (soft fork)
  Title: Hashrate Escrows (Consensus layer)
  Author: Paul Sztorc <truthcoin@gmail.com>
          CryptAxe <cryptaxe@gmail.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0300
  Status: Draft
  Type: Standards Track
  Created: 2017-08-14
  License: BSD-2-Clause
  Post-History: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2017-May/014364.html
```

## Abstract

In Bip300, txns are not signed via cryptographic key. Instead, they are
"signed" by hashpower, over time. Like a big multisig, 13150-of-26300,
where each block is a new "signature".

Bip300 emphasizes slow, transparent, auditable transactions which are
easy for honest users to get right and very hard for dishonest users to
abuse. The chief design goal for Bip300 is *partitioning* -- users may
safely ignore Bip300 txns if they want to (or Bip300 entirely).

See [this site](http://www.drivechain.info/) for more information.

## Motivation

As Reid Hoffman [wrote
in 2014](https://blockstream.com/2015/01/13/en-reid-hoffman-on-the-future-of-the-bitcoin-ecosystem/):
"Sidechains allow developers to add features and functionality to the
Bitcoin universe without actually modifying the Bitcoin Core
code...Consequently, innovation can occur faster, in more flexible and
distributed ways, without losing the synergies of a common platform with
a single currency."

Today, coins such as Namecoin, Monero, ZCash, and Sia, offer features
that Bitcoiners cannot access -- not without selling their BTC to invest
in a rival monetary unit. According to
[coinmarketcap.com](https://coinmarketcap.com/charts/#dominance-percentage),
there is now more value \*outside\* the BTC protocol than within it.
According to [cryptofees.info](https://cryptofees.info/), 15x more txn
fees are paid outside the BTC protocol, than within it.

Software improvements to Bitcoin rely on developer consensus -- BTC will
pass on a good idea if it is even slightly controversial. Development is
slow: we are now averaging one major feature every 5 years.

Sidechains allow for competitive "benevolent dictators" to create a new
sidechain at any time. These dictators are accountable only to their
users, and (crucially) they are protected from rival dictators. Users
can move their BTC among these different pieces of software, as \*they\*
see fit.

BTC can copy every useful technology, as soon as it is invented;
scamcoins lose their justification and become obsolete; and the
community can be pro-creativity, knowing that Layer1 is protected from
harmful changes.

## Specification

### Overview

Bip300 allows for six new blockchain messages (these have consensus
significance):

  - M1. "Propose New Sidechain"
  - M2. "ACK Proposal"
  - M3. "Propose Bundle"
  - M4. "ACK Bundle"
  - M5. Deposit -- a transfer of BTC from-main-to-side
  - M6. Withdrawal -- a transfer of BTC from-side-to-main

Nodes organize those messages into two caches:

  - D1. "The Sidechain List", which tracks the 256 Hashrate Escrows
    (Escrows are slots that a sidechain can live in).
  - D2. "The Withdrawal List", which tracks the withdrawal-Bundles
    (coins leaving a Sidechain).

#### D1 (The Sidechain List)

D1 is a list of active sidechains. D1 is updated via M1 and M2.

<table>
<thead>
<tr class="header">
<th><p>Field No.</p></th>
<th><p>Label</p></th>
<th><p>Type</p></th>
<th><p>Description / Purpose</p></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td><p>1</p></td>
<td><p>Escrow Number</p></td>
<td><p>uint8_t</p></td>
<td><p>The escrow's ID number. Used to uniquely refer to each sidechain.</p></td>
</tr>
<tr class="even">
<td><p>2</p></td>
<td><p>Version</p></td>
<td><p>int32_t</p></td>
<td><p>Version number.</p></td>
</tr>
<tr class="odd">
<td><p>3</p></td>
<td><p>String KeyID</p></td>
<td><p>string</p></td>
<td><p>Used to derive all sidechain deposit addresses.</p></td>
</tr>
<tr class="even">
<td><p>4<br />
</p></td>
<td><p>Sidechain Private Key</p></td>
<td><p>string</p></td>
<td><p>The private key of the sidechain deposit script.</p></td>
</tr>
<tr class="odd">
<td><p>5<br />
</p></td>
<td><p>ScriptPubKey</p></td>
<td><p>CScript</p></td>
<td><p>Where the sidechain coins go. This always stays the same, even though the CTIP (UTXO) containing the coins is always changing.</p></td>
</tr>
<tr class="even">
<td><p>6</p></td>
<td><p>Sidechain Name</p></td>
<td><p>string</p></td>
<td><p>A human-readable name of the sidechain.</p></td>
</tr>
<tr class="odd">
<td><p>7</p></td>
<td><p>Sidechain Description</p></td>
<td><p>string</p></td>
<td><p>A human-readable name description of the sidechain.</p></td>
</tr>
<tr class="even">
<td><p>8</p></td>
<td><p>Hash1 - tarball hash</p></td>
<td><p>uint256</p></td>
<td><p>Intended as the sha256 hash of the tar.gz of the canonical sidechain software. (This is not enforced anywhere by Bip300, and is for human purposes only.)</p></td>
</tr>
<tr class="odd">
<td><p>9</p></td>
<td><p>Hash2 - git commit hash</p></td>
<td><p>uint160</p></td>
<td><p>Intended as the git commit hash of the canonical sidechain node software. (This is not enforced anywhere by Bip300, and is for human purposes only.)</p></td>
</tr>
<tr class="even">
<td><p>10</p></td>
<td><p>Active</p></td>
<td><p>bool</p></td>
<td><p>Does this sidechain slot contain an active sidechain?<br />
</p></td>
</tr>
<tr class="odd">
<td><p>11</p></td>
<td><p>"CTIP" -- Part 1 "TxID"</p></td>
<td><p>uint256</p></td>
<td><p>The CTIP, or "Critical (TxID, Index) Pair" is a variable for keeping track of where the sidechain's money is (ie, which member of the UTXO set).</p></td>
</tr>
<tr class="even">
<td><p>12</p></td>
<td><p>"CTIP" -- Part 2 "Index"</p></td>
<td><p>int32_t</p></td>
<td><p>Of the CTIP, the second element of the pair: the Index. See #11 above.</p></td>
</tr>
</tbody>
</table>

#### D2 (The Withdrawal List)

D2 lists withdrawal-attempts. If these attempts succeed, they will pay
coins "from" a Bip300-locked UTXO, to new UTXOs controlled by the
withdrawing-user. Each attempt pays out many users, so we call these
withdrawal-attempts "Bundles".

D2 is driven by M3, M4, M5, and M6. Those messages enforce the following
principles:

1.  The Bundles have a canonical order (first come first serve).
2.  From one block to the next, every "Blocks Remaining" field decreases
    by 1.
3.  When "Blocks Remaining" reaches zero the Bundle is removed.
4.  From one block to the next, the value in "ACKs" may either increase
    or decrease, by a maximum of 1 (see M4).
5.  If a Bundle's "ACKs" reach 13150 or greater, it "succeeds" and its
    corresponding M6 message can be included in a block.
6.  If the M6 of a Bundle is paid out, it is also removed.
7.  If a Bundle cannot possibly succeed ( 13500 - "ACKs" \> "Blocks
    Remaining" ), it is removed immediately.

| Field No. | Label                  | Type      | Description / Purpose                                                                                                                                                                                  |
| --------- | ---------------------- | --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1         | Sidechain Number       | uint8\_t  | Links the withdrawal-request to a specific hashrate escrow.                                                                                                                                            |
| 2         | Bundle Hash            | uint256   | A withdrawal attempt. Specifically, it is a "blinded transaction id" (ie, the double-Sha256 of a txn that has had two fields zeroed out, see M6) of a txn which could withdraw funds from a sidechain. |
| 3         | ACKs (Work Score)      | uint16\_t | The current ACK-counter, which is the total number of ACKs (the PoW that has been used to validate the Bundle).                                                                                        |
| 4         | Blocks Remaining (Age) | uint16\_t | The number of blocks which this Bundle has remaining to accumulate ACKs                                                                                                                                |

### The Six New Bip300 Messages

First, how are new sidechains created?

They are first proposed (with M1), and later acked (with M2). This
process resembles Bip9 soft fork activation.

#### M1 -- Propose Sidechain

M1 is a coinbase OP Return output containing the following:

`   1-byte - OP_RETURN (0x6a)`  
`   4-byte - Message header (0xD5E0C4AF)`  
`   N-byte - The serialization of the sidechain.`  
`     1-byte nSidechain`  
`     4-byte nVersion`  
`     x-byte strKeyID`  
`     x-byte strPrivKey`  
`     x-byte scriptPubKey`  
`     x-byte title`  
`     x-byte description`  
`     32-byte hashID1`  
`     20-byte hashID2`

##### Examples

<img src="bip-0300/m1-gui.jpg?raw=true" align="middle"></img>

<img src="bip-0300/m1-cli.png?raw=true" align="middle"></img>

#### M2 -- ACK Sidechain Proposal

M2 is a coinbase OP Return output containing the following:

`   1-byte - OP_RETURN (0x6a)`  
`   4-byte - Message header (0xD6E1C5BF)`  
`   32-byte - sha256D hash of sidechain's serialization`

##### Notes

The new M1/M2 validation rules are:

1.  Any miner can propose a new sidechain (M1) at any time. This
    procedure resembles BIP 9 soft fork activation: the network must see
    a properly-formatted M1, followed by "acknowledgment" of the
    sidechain (M2) in 90% of the following 2016 blocks.
2.  Bip300 comes with only 256 sidechain-slots. If all are used, it is
    possible to "overwrite" a sidechain. This requires vastly more M2
    ACKs -- 50% of the following 26300 blocks must contain an M2. The
    possibility of overwrite, does not change the Bip300 security
    assumptions (because we already assume that the sidechain is
    vulnerable to miners, at a rate of 1 catastrophe per 13150 blocks).

#### Notes on Withdrawing Coins

Bip300 withdrawals ("M6") are very significant.

For an M6 to be valid, it must be first "prepped" by one M3 and then
13,150+ M4s. M3 and M4 are about "Bundles".

##### What are Bundles?

Sidechain withdrawals take the form of “Bundles” -- named because they
"bundle up" many individual withdrawal-requests into a single rare
layer1 transaction.

Sidechain full nodes aggregate the withdrawal-requests into a big set.
The sidechain calculates what M6 would have to look like, to pay all of
these withdrawal-requests out. Finally, the sidechain calculates what
the hash of this M6 would be. This 32-byte hash identifies the Bundle.

This 32-byte hash is what miners will be slowly ACKing over 3-6 months,
not the M6 itself (nor any sidechain data, of course).

A bundle either pays all its withdrawals out (via M6), or else it fails
(and pays nothing out).

\===== Bundle Hash = Blinded TxID of M6 =====

The Bundle hash is static as it is being ACKed. Unfortunately, the M6
TxID will be constantly changing -- as users deposit to the sidechain,
the input to M6 will change.

To solve this problem, we do something conceptually similar to
AnyPrevOut (BIP 118). We define a "blinded TxID" as a way of hashing a
txn, in which some bytes are first overwritten with zeros. These are:
the first input and the first output. Via the former, a sidechain can
accept deposits, even if we are acking a TxID that spends from it later.
Via the latter, we can force all of the non-withdrawn coins to be
returned to the sidechain (even if we don't yet know how many coins this
will be).

#### M3 -- Propose Bundle

M3 is a coinbase OP Return output containing the following:

`   1-byte - OP_RETURN (0x6a)`  
`   4-byte - Commitment header (0xD45AA943)`  
`   32-byte - The Bundle hash, to populate a new D2 entry`

The new validation rules pertaining to M3 are:

1.  If the network detects a properly-formatted M3, it must add an entry
    to D2 in the very next block. The starting "Blocks Remaining" value
    is 26,299. The starting ACKs count is 1.
2.  Each block can only contain one M3 per sidechain.

Once a Bundle is in D2, how can we give it enough ACKs to make it valid?

#### M4 -- ACK Bundle(s)

M4 is a coinbase OP Return output containing the following:

`   1-byte - OP_RETURN (0x6a)`  
`   4-byte - Commitment header (0xD77D1776)`  
`   1-byte - Version`  
`   n-byte - The vector describing the "upvoted" bundle-choice, for each sidechain.`

Version 0x01 uses one byte per sidechain, and applies in most cases.
Version 0x02 uses two bytes per sidechain and applies in unusual
situations where at least one sidechain has more than 256 distinct
withdrawal-bundles in progress at one time. Other interesting versions
are possible: 0x03 might say "do exactly what was done in the previous
block" (which could consume a fixed 6 bytes total, regardless of how
many sidechains). 0x04 might say "upvote everyone who is clearly in the
lead" (which also would require a mere 6 bytes), and so forth.

If a sidechain has no pending bundles, then it is skipped over when M4
is created and parsed.

The upvote vector will code "abstain" as 0xFF (or 0xFFFF); it will code
"alarm" as 0xFE (or 0xFFFE). Otherwise it simply indicates which
withdrawal-bundle in the list, is the one to be "upvoted". For example,
if there are two sidechains, and we wish to upvote the 7th bundle on
sidechain \#1 plus the 4th bundle on sidechain \#2, then the vector
would be 0x0704.

The M4 message will be invalid (and invalidate the block), if it tries
to upvote a Bundle that doesn't exist (for example, trying to upvote the
7th bundle on sidechain \#2, when sidechain \#2 has only three bundles).
If there are no Bundles at all (no one is trying to withdraw from any
sidechain), then \*any\* M4 message present in the coinbase will be
invalid. If M4 is NOT present in a block, then it is treated as
"abstain".

The ACKed withdrawal will gain one point for its ACK field. Therefore,
the ACK-counter of any Bundle can only change by (-1,0,+1).

Within a sidechain-group, upvoting one Bundle ("+1") requires you to
downvote all other Bundles in that group. However, the minimum
ACK-counter is zero. While only one Bundle can be upvoted at once; the
whole group can all be unchanged at once ("abstain"), and they can all
be downvoted at once ("alarm").

Finally, we describe Deposits and Withdrawals.

#### M5 -- Deposit BTC to Sidechain

Both M5 and M6 are regular Bitcoin txns. They are distinguished from
regular txns (non-M5 non-M6 txns), when they select one of the special
Bip300 CTIP UTXOs as one of their inputs (see D1).

All of a sidechain’s coins, are stored in one UTXO, called the "CTIP".
Every time a deposit or withdrawal is made, the CTIP changes. Each
deposit/withdrawal will select the sidechains CTIP, and generate a new
CTIP. (Deposits/Withdrawals never cause UTXO bloat.) The current CTIP is
cached in D1 (above).

If the **quantity of coins**, in the from-CTIP-to-CTIP transaction, goes
**up**, (ie, if the user is adding coins), then the txn is treated as a
Deposit (M5). Else it is treated as a Withdrawal (M6). See
[here](https://github.com/drivechain-project/mainchain/blob/e37b008fafe0701b8313993c8b02ba41dc0f8a29/src/validation.cpp#L667-L780).

As far as mainchain consensus is concerned, all deposits to a sidechain
are always valid.

#### M6 -- Withdraw BTC from a Sidechain

We come, finally, to the critical matter: where users can take their
money \*out\* of the sidechain.

First, M6 must obey the same CTIP rules of M5 (see immediately above).

Second, an M6 is only valid for inclusion in a block, if its blinded
TxID matches an "approved" Bundle hash (ie, one with an ACK score of
13150+). In other words, an M6 can only be included in a block, after
the 3+ month (13150 block) ceremony.

Third, M6 must meet two accounting criteria, lest it be invalid:

1.  "Give change back to Escrow" -- The first output, TxOut0, must be
    paid back to the sidechain's Bip300 script. In other words, all
    non-withdrawn coins must be paid back into the sidechain.
2.  "No traditional txn fee" -- For this txn, the sum of all inputs must
    equal the sum of all outputs. No traditional tx fee is possible. (Of
    course, there is still a txn fee for miners: it is paid via an OP
    TRUE output in the Bundle.) We want the withdraw-ers to set the fee
    "inside" the Bundle, and ACK it over 3 months like everything else.

## Backward compatibility

As a soft fork, older software will continue to operate without
modification. Non-upgraded nodes will see a number of phenomena that
they don't understand -- coinbase txns with non-txn data, value
accumulating in anyone-can-spend UTXOs for months at a time, and then
random amounts leaving these UTXOs in single, infrequent bursts.
However, these phenomena don't affect them, or the validity of the money
that they receive.

( As a nice bonus, note that the sidechains themselves inherit a
resistance to hard forks. The only way to guarantee that all different
sidechain-nodes will always report the same Bundle, is to upgrade
sidechains via soft forks of themselves. )

## Deployment

This BIP will be deployed via UASF-style block height activation. Block
height TBD.

## Reference Implementation

See: <https://github.com/drivechain-project/mainchain>

Also, for interest, see an example sidechain here:
<https://github.com/drivechain-project/sidechains/tree/testchain>

## References

<https://github.com/drivechain-project/mainchain>
<https://github.com/drivechain-project/sidechains/tree/testchain> See
<http://www.drivechain.info/literature/index.html>

## Credits

Thanks to everyone who contributed to the discussion, especially:
ZmnSCPxj, Adam Back, Peter Todd, Dan Anderson, Sergio Demian Lerner,
Chris Stewart, Matt Corallo, Sjors Provoost, Tier Nolan, Erik Aronesty,
Jason Dreyzehner, Joe Miyamoto, Ben Goldhaber.

## Copyright

This BIP is licensed under the BSD 2-clause license.
