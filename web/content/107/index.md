+++
title = "Dynamic limit on the block size"
date = 2015-09-11
weight = 107
in_search_index = true

[taxonomies]
authors = ["Washington Y. Sanchez"]
status = ["Rejected"]

[extra]
bip = 107
status = ["Rejected"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0107.mediawiki"
+++

``` 
  BIP: 107
  Layer: Consensus (hard fork)
  Title: Dynamic limit on the block size
  Author: Washington Y. Sanchez <washington.sanchez@gmail.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0107
  Status: Rejected
  Type: Standards Track
  Created: 2015-09-11
  License: PD
```

## Abstract

This BIP proposes a dynamic limit to the block size based on transaction
volume.

## Motivation

Over the next few years, large infrastructure investments will be made
into:

1.  Improving global network connectivity
2.  Improving block propagation across the Bitcoin network
3.  Layer 2 services and networks for off-chain transactions
4.  General efficiency improvements to transactions and the blockchain

<!-- end list -->

  - While there is a consensus between Bitcoin developers, miners,
    businesses and users that the block size needs to be increased,
    there is a lingering concern over the potential unintended
    consequences that may augment the trend towards network and mining
    centralization (largely driven by mining hardware such as ASICs) and
    thereby threaten the security of the network.
  - In contrast, failing to respond to elevated on-chain transaction
    volume may lead to a consumer-failure of Bitcoin, where ordinary
    users - having enjoyed over 6 years of submitting transactions
    on-chain at relatively low cost - will be priced out of blockchain
    with the emergence of a prohibitive 'fee market'.
  - These two concerns must be delicately balanced so that all users can
    benefit from a robust, scalable, and neutral network.

## Specification

  - Increases in the block size will occur in 2 phases
  - **Phase 1**
      - The block size will be increased similar to [Adam Back's
        proposal](https://twitter.com/adam3us/status/636410827969421312 "wikilink"),
        as a safe runway prior to switching to Phase 2, while network
        and protocol infrastructure is improved
      - The schedule:
          - *2016-2017:* 2 MB
          - *2018-2019:* 4 MB
          - *2020:* 6 MB
  - **Phase 2**
      - In 2020, the maximum block size will be increased dynamically
        according to sustained increases in transaction volume
      - Every 4032 blocks (\~4 weeks), a CHECK will be performed to
        determine if a raise in the maximum block size should occur
          - This calculates to a theoretical maximum of 13 increases per
            year
      - IF of the last \>= 3025 blocks were \>=60% full, the maximum
        block size will be increased by 10%
      - The maximum block size can only ever be increased, not decreased
  - The default `limitfreerelay` will also be raised in proportion to
    maximum block size increases
      - Transactions without fees can continue to be submitted and
        relayed on the network as per normal
      - `limitfreerelay` also helps counter attempts to trigger a block
        size increase by 'penny-flooding'

For example:

  - When the dynamic rules for increasing the block size go live on
    January 1st 2020, the starting maximum block size will be 6 MB
  - IF \>=3025 blocks are \>= 3.6 MB, the new maximum block size become
    6.6 MB.
  - The theoretical maximum block size at the end of 2020 would be
    \~20.7 MB, assuming all 13 increases are triggered every 4 weeks by
    the end of the year.

## Rationale

  - **Phase 1**
      - This runway has a schedule for conservative increases to the
        block size in order to relieve transaction volume pressure while
        allowing network and protocol infrastructure improvements to be
        made, as well as layer 2 services to emerge
  - **Phase 2**
      - Why 60% full blocks?
          - IF blocks are 60% full, they count as a *vote* towards
            increasing the block size
          - If this parameter is too low, the trigger sensitivity may be
            too high and vulnerable to spam attacks or miner collusion
          - Setting the parameter too high may set the trigger
            sensitivity too low, causing transaction delays that are
            trying to be avoided in the first place
          - Between September 2013-2015, the standard deviation measured
            from average block size (n=730 data points from
            blockchain.info) was \~ 0.13 MB or 13% of the maximum block
            size
              - If blocks needed to be 90% full before an increase were
                triggered, normal variance in the average block size
                would mean some blocks would be full before an increase
                could be triggered
          - Therefore, we need a *safe distance* away from the maximum
            block size to avoid normal block size variance hitting the
            limit. The 60% level represents a 3 standard deviation
            distance from the limit.
      - Why 3025 blocks?
          - The assessment period is 4032 blocks or \~ 4 weeks, with the
            threshold set as 4032 blocks/0.75 + 1
          - Increases in the maximum block size should only occur after
            a sustained trend can be observed in order to:
            1.  Demonstrate a market-driven secular elevation in the
                transaction volume
            2.  Increase the cost to trigger an increase by spam attacks
                or miner collusion with zero fee transactions
          - In other words, increases to the maximum block size must be
            conservative but meaningful to relieve transaction volume
            pressure in response to true market demand
      - Why 10% increase in the block size?
          - Increases in the block size are designed to be conservative
            and in balance with the number of theoretical opportunities
            to increase the block size per year
          - Makes any resources spent for spam attacks or miner
            collusion relatively expensive to achieve a minor increase
            in the block size. A sustained attack would need to be
            launched that may be too costly, and ideally detectable by
            the community

## Deployment

Similar deployment model to BIP101:

> Activation is achieved when 750 of 1,000 consecutive blocks in the
> best chain have a version number with the first, second, third, and
> thirtieth bits set (0x20000007 in hex). The activation time will be
> the timestamp of the 750'th block plus a two week (1,209,600 second)
> grace period to give any remaining miners or services time to upgrade
> to support larger blocks.

## Acknowledgements

Thanks to Austin Williams, Brian Hoffman, Angel Leon, Bulukani Mlalazi,
Chris Pacia, and Ryan Shea for their comments.

## Copyright

This work is placed in the public domain.
