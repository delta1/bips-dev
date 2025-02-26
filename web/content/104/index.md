+++
title = "Block75 - Max block size like difficulty"
date = 2017-01-13
weight = 104
in_search_index = true

[taxonomies]
authors = ["t.khan"]
status = ["Rejected"]

[extra]
bip = 104
status = ["Rejected"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0104.mediawiki"
+++

``` 
  BIP: 104
  Layer: Consensus (hard fork)
  Title: 'Block75' - Max block size like difficulty
  Author: t.khan <teekhan42@gmail.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0104
  Status: Rejected
  Type: Standards Track
  Created: 2017-01-13
  License: BSD-2-Clause
           GNU-All-Permissive
```

## Abstract

Automatic adjustment of max block size with the target of keeping blocks
75% full, based on the average block size of the previous 2016 blocks.
This would be done on the same schedule as difficulty.

## Motivation

Blocks are already too full and cannot support further transaction
growth. While SegWit and Lightning (and other off-chain solutions) will
help, they will not solve this problem.

Bitcoin needs a reasonably effective and predictable way of managing the
maximum block size which allows moderate growth, keeps max block size as
small as possible, and prevents wild swings in transaction fees.

The every two-week and automatic adjustment of difficulty has proven to
be a reasonably effective and predictable way of managing how quickly
blocks are mined. It works well because humans aren’t involved (except
for setting the original target of a 10 minute per block average), and
therefore it isn’t political or contentious. It’s simply a response to
changing network resources.

It’s clear at this point that human beings should not be involved in the
determination of max block size, just as they’re not involved in
deciding the difficulty. Therefore, it is logical and consistent with
Bitcoin’s design to implement a permanent solution which, as with the
difficulty adjustment, is simply an automatic response to changing
transaction volumes. With the target of keeping blocks 75% full on
average, this is the goal of Block75.

## Specification

The max block size will be recalculated every 2016 blocks, along with
difficulty, using Block75’s simple algorithm:

`new max block size = x + (x * (AVERAGE_CAPACITY - TARGET_CAPACITY))`

  - TARGET\_CAPACITY = 0.75    //Block75's target of keeping blocks 75%
    full
  - AVERAGE\_CAPACITY = average percentage full of the last 2016 blocks,
    as a decimal
  - x = current max block size

All code which generates/validates blocks or uses/references the current
hardcoded limits will need to be changed to support Block75.

## Rationale

The 75% full block target was selected because:

  - it is the middle ground between blocks being too small (average 100%
    full) and blocks being unnecessarily large (average 50% full)
  - it can handle short-term spikes in transaction volume of up to 33%
  - it limits the growth of max block size to less than 25% over the
    previous period
  - it will maintain average transaction fees at a stable level similar
    to that of May/June 2016

The 2016 block (\~2 weeks) period was selected because:

  - it has been shown to be reasonably adaptive to changing network
    resources (re: difficulty)
  - the frequent and gradual adjustments that result will be relatively
    easy for miners and node operators to predict and adapt to, as any
    unforeseen consequences will be visible well in advance
  - it minimizes any effect a malicious party could have in an attempt
    to manipulate max block size

The Block75 algorithm will adjust the max block size up and down in
response to transaction volume, including changes brought on by SegWit
and Lightning. This is important as it will keep average transaction
fees stable, thereby allowing miners and businesses using Bitcoin more
certainty regarding future income/expenses.

## Other solutions considered

A hardcoded increase to max block size (2MB, 8MB, etc.), rejected
because:

  - only a temporary solution, whatever limit was chosen would
    inevitably become a problem again
  - would cause transaction fees to vary wildly over time

Allow miners to vote for max block size, rejected because:

  - overly complex and political
  - human involvement makes this slow to respond to changing transaction
    volumes
  - focuses power over max block size to a relatively small group of
    people
  - unpredictable transaction fees caused by this would create
    uncertainty in the ecosystem

## Backward Compatibility

This BIP is not backward compatible (hard fork). Any code which fully
validates blocks must be upgraded prior to activation, as failure to do
so will result in rejection of blocks over the current 1MB limit.

## Activation

To help negate some of the risks associated with a hard fork and to
prevent a single relatively small mining pool from preventing Block75's
adoption, activation would occur at the next difficulty adjustment once
900 of the last 1,000 blocks mined signal support and a grace period of
4,032 blocks (\~1 month) has elapsed.

## Copyright

This BIP is dual-licensed under the BSD 2-clause license and the GNU
All-Permissive License.
