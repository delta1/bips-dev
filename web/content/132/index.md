+++
title = "Committee-based BIP Acceptance Process"
date = 2015-08-31
weight = 132
in_search_index = true

[taxonomies]
authors = ["Andy Chase"]
status = ["Withdrawn"]

[extra]
bip = 132
status = ["Withdrawn"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0132.mediawiki"
+++

``` 
  BIP: 132
  Title: Committee-based BIP Acceptance Process
  Author: Andy Chase <theandychase@gmail.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0132
  Status: Withdrawn
  Type: Process
  Created: 2015-08-31
  License: PD
```

## Abstract

The current process for accepting a BIP is not clearly defined. While
[BIP-0001](https://github.com/bitcoin/bips/blob/master/bip-0001.mediawiki)
defines the process for writing and submitting a Bitcoin Improvement
Proposal to the community it does not specify the precise method for
which BIPs are considered accepted or rejected.

This proposal sets up a method for determining BIP acceptance.

This BIP has two parts:

  - It sets up a **process** which a BIP goes through for comments and
    acceptance. The Process is:
      - BIP Draft
      - Submitted for comments (2 weeks)
      - Waiting on opinion (2 weeks)
      - BIP becomes either Accepted or Deferred
  - It sets up **committees** for reviewing comments and indicating
    acceptance under precise conditions.
      - Committees are authorized groups that represent client authors,
        miners, merchants, and users (each as a segment). Each one must
        represent at least 1% stake in the Bitcoin ecosystem.

BIP acceptance is defined as at least 70% of the represented percentage
stake in 3 out of the 4 Bitcoin segments.

## Copyright

This document is placed into the public domain.

## Motivation

BIPs represent important improvements to Bitcoin infrastructure, and in
order to foster continued innovation, the BIP process must have clearly
defined stages and acceptance acknowledgement.

## Rationale

A committee system is used to organize the essential concerns of each
segment of the Bitcoin ecosystem. Although each segment may have many
different viewpoints on each BIP, in order to seek a decisive yes/no on
a BIP, a representational authoritative structure is sought. This
structure should be fluid, allowing people to move away from committees
that do not reflect their views and should be re-validated on each BIP
evaluation.

## Weaknesses

Each committee submits a declaration including their claim to represent
a certain percentage of the Bitcoin ecosystem in some way. Though
guidelines are given, it's up to each committee to prove their stake,
and it's up to the reader of the opinions to decide if a BIP was truly
accepted or rejected.

The author doesn't believe this is a problem because a BIP cannot be
forced on client authors, miners, merchants, or users. Ultimately this
BIP is a tool for determining whether a BIP is overwhelmingly accepted.
If one committee's validity claim becomes the factor that decides
whether the BIP will succeed or fail, this process simply didn't return
a clear answer and the BIP should be considered deferred.

## Process

  - **Submit for Comments.** The first BIP champion named in the
    proposal can call a "submit for comments" at any time by posting to
    the [Dev Mailing
    List](https://lists.linuxfoundation.org/mailman/listinfo/bitcoin-dev)
    mailling with the BIP number and a statement that the champion
    intends to immediately submit the BIP for comments.
      - The BIP must have been assigned BIP-number (i.e. been approved
        by the BIP editor) to be submitted for comments.
  - **Comments.**
      - After a BIP has been submitted for comments, a two-week waiting
        period begins in which the community should transition from
        making suggestions about a proposal to publishing their opinions
        or concerns on the proposal.
  - **Reported Opinions.**
      - After the waiting period has past, committees must submit a
        summary of the comments which they have received from their
        represented communities.
      - The deadline for this opinion is four weeks after the BIP was
        submitted for comments.
      - Committees cannot reverse their decision after the deadline, but
        at their request may flag their decision as "likely to change if
        another submit for comments is called". Committees can change
        their decision if a resubmit is called.
      - Opinions must include:
          - One of the following statements: "Intend to accept", "Intent
            to implement", "Decline to accept", "Intend to accept, but
            decline to implement".
          - If rejected, the opinion must cite clear and specific
            reasons for rejecting including a checklist for what must
            happen or be change for their committee to accept the
            proposal.
          - If accepted, the committee must list why they accepted the
            proposal and also include concerns they have or what about
            the BIP that, if things changed, would cause the committee
            to likely reverse their decision if another submit for
            comments was called.
  - **Accepted.**
      - If at least 70% of the represented percentage stake in 3 out of
        4 segments accept a proposal, the BIP is considered accepted.
      - If a committee fails to submit an opinion, consider the opinion
        "Decline to accept".
      - The BIP cannot be substantially changed at this point, but can
        be replaced. Minor changes or clarifications are allowed but
        must be recorded in the document.
  - **Deferred.**
      - If the acceptance test above is not met, the BIP is sent back
        into suggestions.
      - BIP can be modified and re-submitted for a comments no sooner
        than two months after the date of the previous submit for
        comments is called.
      - The BIP is marked rejected after two failed submission attempts.
        A rejected BIP can still be modified and re-submitted.

## Committees

**BIP Committees.**

  - BIP Committees are representational structures that represent
    critical segments of the Bitcoin ecosystem.
  - Each committee must prove and maintain a clear claim that they
    represent at least 1% of the Bitcoin ecosystem in some form.
  - If an organization or community does not meet that requirement, it
    should conglomerate itself with other communities and organizations
    so that it does.
  - The segments that committees can be based around are:
      - Bitcoin software
      - Exchanges/Merchants/services/payment processors
      - Mining operators
      - User communities
  - A person may be represented by any number of segments, but a
    committee cannot re-use the same resource as another committee in
    the same segment.

**Committee Declarations.**

  - At any point, a Committee Declaration can be posted.
  - This Declaration must contain details about:
      - The segment the Committee is representing
      - Who the committee claim to represent and it's compositional
        makeup (if made up of multiple miner orgs, user orgs, companies,
        clients, etc).
      - Proof of claim and minimum 1% stake via:
          - Software: proof of ownership and user base (Min 1% of
            Bitcoin userbase)
          - Merchant: proof of economic activity (Min 1% of Bitcoin
            economic activity)
          - Mining: proof of work (Min 1% of Hashpower)
          - For a user organization, auditable signatures qualifies for
            a valid committee (Min 1% of Bitcoin userbase)
      - Who is running the committee, their names and roles
      - How represented members can submit comments to the committee
      - A code of conduct and code of ethics which the committee
        promises to abide by
  - A committee declaration is accepted if:
      - The declaration includes all of the required elements
      - The stake is considered valid
      - Committee validation is considered when considering the results
        of opinions submitted by committee on a BIP. A committee must
        have met the required stake percentage before a BIP is submitted
        for comments, and have maintained that stake until a valid
        opinion is submitted.
  - Committees can dissolve at any time or submit a declaration at any
    time
  - Declaration must have been submitted no later than the third day
    following a BIP's request for comments to be eligible for inclusion
    in a BIP

## BIP Process Management Role

BIPs, Opinions, and Committee Declaration must be public at all times.

A BIP Process Manager should be chosen who is in charge of:

  - Declaring where and how BIPs, Opinions, and Committee Declaration
    should be posted and updated officially.
  - Maintaining the security and authenticity of BIPs, Opinions, and
    Committee Declarations
  - Publishing advisory documents about what kinds of proof of stakes
    are valid and what kinds should be rejected.
  - Naming a series of successors for the roles of the BIP Process
    Manager and BIP Editor (BIP-001) as needed.

## Conditions for activation

In order for this process BIP to become active, it must succeed by its
own rules. At least a 4% sample of the Bitcoin community must be
represented, with at least one committee in each segment included. Once
at least one committee has submitted a declaration, a request for
comments will be called and the process should be completed from there.
