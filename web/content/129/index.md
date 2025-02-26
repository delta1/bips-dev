+++
title = "Bitcoin Secure Multisig Setup (BSMS)"
date = 2020-11-10
weight = 129
in_search_index = true

[taxonomies]
authors = ["Hugo Nguyen", "Peter Gray", "Marko Bencun", "Aaron Chen", "Rodolfo Novak"]
status = ["Proposed"]

[extra]
bip = 129
status = ["Proposed"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0129.mediawiki"
+++

``` 
  BIP: 129
  Layer: Applications
  Title: Bitcoin Secure Multisig Setup (BSMS)
  Author: Hugo Nguyen <hugo@nunchuk.io>
          Peter Gray <peter@coinkite.com>
          Marko Bencun <marko@shiftcrypto.ch>
          Aaron Chen <aarondongchen@gmail.com>
          Rodolfo Novak <rodolfo@coinkite.com>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0129
  Status: Proposed
  Type: Standards Track
  Created: 2020-11-10
  License: BSD-2-Clause
```

## Introduction

### Abstract

This document proposes a mechanism to set up multisig wallets securely.

### Copyright

This BIP is licensed under the 2-clause BSD license.

### Motivation

The Bitcoin multisig experience has been greatly streamlined under
[BIP-0174 (Partially Signed Bitcoin
Transaction)](https://github.com/bitcoin/bips/blob/master/bip-0174.mediawiki).
However, what is still missing is a standardized process for setting up
multisig wallets securely across different vendors.

There are a number of concerns when it comes to setting up a multisig
wallet:

1.  Whether the multisig configuration, such as Signer membership,
    script type, derivation paths and number of signatures required, is
    correct and not tampered with.
2.  Whether the keys or the multisig configuration are leaked during the
    setup.
3.  Whether the Signer persists the multisig configuration in their
    respective storage, and under what format.
4.  Whether the Signer's storage is tamper-proof.
5.  Whether the Signer subsequently uses the multisig configuration to
    generate and verify receive and change addresses.

An attacker who can modify the multisig configuration can steal or hold
funds for ransom by duping the user into sending funds to the wrong
address. An attacker who cannot modify the configuration but can learn
about the keys and/or the configuration can monitor transactions in the
wallet, resulting in loss of privacy.

This proposal seeks to address concerns \#1, \#2 and \#3: to mitigate
the risk of tampering during the initial setup phase, and to define an
interoperable multisig configuration format.

Concerns \#4 and \#5 should be handled by Signers and are out of scope
of this proposal.

## Specification

### Prerequisites

This proposal assumes the parties in the multisig support
[BIP-0032](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki),
[BIP-0322](https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki),
[the descriptor
language](https://github.com/bitcoin/bitcoin/blob/master/doc/descriptors.md)
and [AES encryption](https://tools.ietf.org/html/rfc3686).

### File Extensions

All descriptor and key records should have a `.bsms` file extension.
Encrypted data should have a `.dat` extension.

### Roles

#### Coordinator

The Coordinator initiates the multisig setup. The Coordinator determines
what type of multisig is used and the exact policy script. If encryption
is enabled, the Coordinator also distributes a shared secret or shared
secrets to the parties involved for secure communication. The
Coordinator gathers information from the Signers to generate a
descriptor record. The Coordinator distributes the descriptor record
back to the Signers.

#### Signer

The Signer is any software or hardware that controls the private keys
and can sign using those keys. The Signer is a participating member in
the multisig. Its responsibilities include providing its key record --
which contains a public key or an Extended Public Key (XPUB) -- to the
Coordinator, verifying that its `KEY` is included in the descriptor
record and persisting the descriptor record in its storage.

### Setup Process

#### Round 1

##### Coordinator

  - The Coordinator creates a new multisig wallet creation session. The
    Coordinator constructs the multisig script and its policy
    parameters, such as the required number of signatures and the total
    number of Signers (`M` and `N`).
  - The session should expire after some time period determined by the
    Coordinator, e.g., 24 hours. The timeout allows the encryption key
    to have lower entropy.
  - If encryption is enabled, the Coordinator distributes a secret
    `TOKEN` to each Signer over a secure channel. The Signer can use the
    `TOKEN` to derive an `ENCRYPTION_KEY`. Refer to the
    [\#Encryption](#Encryption "wikilink") section below for details on
    the `TOKEN`, the key derivation function and the encryption scheme.
    Depending on the use case, the Coordinator can decide whether to
    share one common `TOKEN` for all Signers, or to have one per Signer.
  - If encryption is disabled, the `TOKEN` is set to `0x00`, and all the
    encryption/decryption steps below can be skipped.

##### Signer

  - The Signer initiates the multisig wallet creation session by setting
    the `TOKEN`. The Signer derives an `ENCRYPTION_KEY` from the
    `TOKEN`. The Signer can keep the session open until a different
    value for the `TOKEN` is set.
  - The Signer generates a key record by prompting the user for a
    multisig derivation path and retrieves the `KEY` at that derivation
    path. Alternatively, the Signer can choose a path on behalf of the
    user. If the Signer chooses the path, it should try to avoid reusing
    `KEY`s for different wallets.
  - The first line in the record must be the specification version
    (`BSMS 1.0` as of this writing). The second line must be the
    hex-encoded `TOKEN`. The third line must be the `KEY`. The `KEY` is
    a public key or an XPUB plus the key origin information, written in
    the descriptor-defined format, i.e.: `[{master key
    fingerprint}/{derivation path}]{KEY}`. The fourth line is a text
    description of the key, 80 characters maximum. The fifth line must
    be a `SIG`, whereas `SIG` is the signature generated by using the
    private key associated with the public key or XPUB to sign the first
    four lines. The signature should follow
    [BIP-0322](https://github.com/bitcoin/bips/blob/master/bip-0322.mediawiki),
    legacy format accepted.
  - The Signer calculates the Message Authentication Code (`MAC`) for
    the record. The first 16 bytes of the `MAC` serves as the
    Initialization Vector (`IV`) for the encryption.
  - The Signer encrypts the key record with the `ENCRYPTION_KEY` and
    `IV`.
  - The Signer encodes the `MAC` and the ciphertext into hexadecimal
    format, then concatenates the results: `(MAC || ciphertext)`.

#### Round 2

##### Coordinator

  - The Coordinator gathers key records from all participating Signers.
    The Coordinator verifies that there are exactly `N` unique key
    records before the wallet setup session expires.
  - For each key record, the Coordinator extracts the `MAC` from the
    data, sets `IV` to the first 16 bytes of the `MAC`, then decrypts
    the ciphertext using the `ENCRYPTION_KEY` and `IV`.
  - The Coordinator verifies that the included `MAC` is valid given the
    plaintext.
  - The Coordinator verifies that the key records have compatible
    specification versions.
  - The Coordinator verifies that the included `SIG` is valid given the
    `KEY`.
  - If all key records look good, the Coordinator fills in all necessary
    information to generate a descriptor record.
  - The first line in the descriptor record must be the specification
    version (`BSMS 1.0` as of this writing). The second line must be a
    descriptor or a descriptor template. The third line must be a
    comma-separated list of derivation path restrictions. The paths must
    start with `/` and use non-hardened derivation. If there are no
    template or restrictions, it must say `No path restrictions`. The
    fourth line must be the wallet's first address. If there are path
    restrictions, use the first address from the first path restriction.
  - The Coordinator calculates the `MAC` for the record. The first 16
    bytes of the `MAC` serves as the `IV` for the encryption..
  - The Coordinator encrypts the descriptor record with the
    `ENCRYPTION_KEY` and `IV`.
  - The Coordinator encodes the `MAC` and the ciphertext into
    hexadecimal format, then concatenates the results: `(MAC ||
    ciphertext)`.
  - The Coordinator sends the encrypted descriptor record to all
    participating Signers.

##### Signer

  - The Signer imports the descriptor record.
  - The Signer extracts the `MAC` from the data, sets `IV` to the first
    16 bytes of the `MAC`, then decrypts the ciphertext using the
    `ENCRYPTION_KEY` (derived from the open session) and `IV`.
  - The Signer verifies that the included `MAC` is valid given the
    plaintext.
  - The Signer verifies that it can support the included specification
    version.
  - The Signer verifies that it can support the descriptor or descriptor
    template.
  - The Signer checks that its `KEY` is included in the descriptor or
    descriptor template, using path and fingerprint information
    provided. The check must perform an exact match on the `KEY`s and
    not using shortcuts such as matching fingerprints, which is trivial
    to spoof.
  - The Signer verifies that it is compatible with the derivation path
    restrictions.
  - The Signer verifies that the wallet's first address is valid.
  - For confirmation, the Signer must display to the user the wallet's
    first address and policy parameters, including, but not limited to:
    the derivation path restrictions, `M`, `N`, and the position(s) of
    the Signer's own `KEY` in the policy script. The total number of
    Signers, `N`, is important to prevent a `KEY` insertion attack. The
    position is important for scripts where `KEY` order matters. When
    applicable, all positions of the `KEY` must be displayed. The full
    descriptor or descriptor template must also be available for review
    upon user request.
  - Parties must check with each other that all Signers have the same
    confirmation (except for the `KEY` positions).
  - If all checks pass, the Signer must persist the descriptor record in
    its storage.

This completes the setup.

### Encryption

#### The Token

We define three modes of encryption.

1.  `NO_ENCRYPTION` : the `TOKEN` is set to `0x00`. Encryption is
    disabled.
2.  `STANDARD` : the `TOKEN` is a 64-bit nonce.
3.  `EXTENDED` : the `TOKEN` is a 128-bit nonce.

The `TOKEN` can be converted to one of these formats:

  - A decimal number (recommended). The number must not exceed the
    maximum value of the nonce.
  - A mnemonic phrase using
    [BIP-0039](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki)
    word list. This would be 6 words in `STANDARD` mode. This encoding
    is not recommended in `EXTENDED` mode as it can result in potential
    confusion between seed mnemonics and `TOKEN` mnemonics.
  - A QR code.
  - Other formats.

The flexibility in the data format allows each Signer to customize the
User Experience based on its respective capabilities.

#### Key Derivation

The key derivation function is
[PBKDF2](https://tools.ietf.org/html/rfc2898), with PRF = SHA512.
Specifically:

`DKey = PBKDF2(PRF, Password, Salt, c, dkLen)`

Whereas:

  - PRF = SHA512
  - Password = "No SPOF"
  - Salt = `TOKEN`
  - c = 2048
  - dkLen = 256
  - DKey = Derived `ENCRYPTION_KEY`

#### Encryption Scheme

The encryption scheme is
[AES-256-CTR](https://tools.ietf.org/html/rfc3686).

`MAC = HMAC-SHA256(HMAC_Key, hex-encoded TOKEN || Data)`

`IV = First 16 bytes of MAC`

`Ciphertext = AES-256-CTR-Encrypt(Plaintext, DKey, IV)`

`Plaintext = AES-256-CTR-Decrypt(Ciphertext, DKey, IV)`

Whereas:

  - DKey = `ENCRYPTION_KEY`
  - HMAC\_Key = SHA256(`ENCRYPTION_KEY`)
  - Data = the plaintext, e.g. the entire key record in round 1 and the
    entire descriptor record in round 2

The `MAC` is to be sent along with the key and descriptor record, as
specified above. Because it is a `MAC` over the entire plaintext, this
is essentially an
[Encrypt-and-MAC](https://en.wikipedia.org/wiki/Authenticated_encryption#Encrypt-and-MAC_\(E&M\))
form of authenticated encryption.

### Descriptor Template

The output descriptor language only supports one-dimensional lists. This
proposal introduces a descriptor template to represent multi-dimensional
lists:

`XPUB/**`

Whereas `/**` can be replaced by any number of derivation path
restrictions.

A descriptor template must be accompanied by derivation path
restrictions. Signers should expand the template into concrete
descriptors by replacing `/**` with the restrictions.

For example, the following template and derivation path restrictions:

  - `wsh(sortedmulti(2,XPUB1/**,XPUB2/**))`
  - `/0/*,/1/*`

Should translate to two concrete descriptors:

  - `wsh(sortedmulti(2,XPUB1/0/*,XPUB2/0/*))`
  - `wsh(sortedmulti(2,XPUB1/1/*,XPUB2/1/*))`

## QR Codes

For signers that use QR codes to transmit data, key and descriptor
records can be converted to QR codes, following [the BCR
standard](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-005-ur.md).

Also refer to [UR Type Definition for BIP44
Accounts](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-015-account.md)
and [UR Type Definition for Bitcoin Output
Descriptors](https://github.com/BlockchainCommons/Research/blob/master/papers/bcr-2020-010-output-desc.md)
for more details.

## Compatibility

This specification is not backwards compatible with existing multisig
implementations.

BSMS is opt-in, meaning existing multisig implementations can continue
working as-is, with the caveat that they are likely to have various
pitfalls. Some of the problems with existing solutions have been
described in the [\#Motivation](#Motivation "wikilink") section.

To comply with this standard, a Signer must be able to persist the
descriptor record in its storage.

To use BSMS for a multisig wallet, the user should wait until all
participating Signers in the multisig have implemented BSMS.

## Security

This proposal introduces two layers of protection. The first one is a
temporary, secret `TOKEN`. The second one is the confirmation of the
wallet's first address.

The `TOKEN` is used to encrypt the two rounds of communication between
the Signer and the Coordinator. A `MAC` is also generated from the
`TOKEN` and plaintext to authenticate the data being exchanged. The
`TOKEN` is only needed during the setup phase, and can be safely
discarded afterwards. It is not recommended to use the same `TOKEN` for
multiple wallet creation sessions.

The wallet's first address, on the other hand, can be used to verify the
integrity of the multisig configuration. An attacker who tampers with
the multisig configuration must also change the wallet's first address.
Parties must check with each other that all Signers confirm to the same
address and policy parameters to reduce the chance of tampering.

## Privacy

Encryption helps improve the privacy of the wallet by avoiding sharing
keys and descriptors in plaintext.

If the parties wish to have stronger privacy, it is recommended to use a
higher number of bits for the `TOKEN`, and to completely erase knowledge
of the `TOKEN` after the multisig wallet has been set up.

## Test Vectors

### Mode: `NO_ENCRYPTION` with Public Keys

#### ROUND 1

  - Coordinator
      - M-of-N: 1/2
      - ADDRESS\_TYPE: NATIVE\_SEGWIT
      - TOKEN: 0x00

<!-- end list -->

  - Signer 1
      - MASTER\_KEY\_FINGERPRINT: 59865f44
      - PRIVATE\_KEY (m/48'/0'/0'/2'):
        L5TXU4SdD9e6QGgBjxeegJKxt4FgATLG1TCnFM8JLyEkFuyHEqNM
      - Public Key (m/48'/0'/0'/2'):
        026d15412460ba0d881c21837bb999233896085a9ed4e5445bd637c10e579768ba
      - Legacy signature
      - `signer_1_key.bsms`:

<!-- end list -->

    BSMS 1.0
    00
    [59865f44/48'/0'/0'/2']026d15412460ba0d881c21837bb999233896085a9ed4e5445bd637c10e579768ba
    Signer 1 key
    H6DXgqkCb353BDPkzppMFpOcdJZlpur0WRetQhIBqSn6DFzoQWBtm+ibP5wERDRNi0bxxev9B+FIvyQWq0s6im4=

  - Signer 2
      - MASTER\_KEY\_FINGERPRINT: b7044ca6
      - PRIVATE\_KEY (m/48'/0'/0'/2'):
        KwT7BZDWjos4JAdfKi8NqF46Kj3rppTwN8KGhPbzmmugiZioFW3r
      - Public Key (m/48'/0'/0'/2'):
        030baf0497ab406ff50cb48b4013abac8a0338758d2fd54cd934927afa57cc2062
      - Legacy signature
      - `signer_2_key.bsms`:

<!-- end list -->

    BSMS 1.0
    00
    [b7044ca6/48'/0'/0'/2']030baf0497ab406ff50cb48b4013abac8a0338758d2fd54cd934927afa57cc2062
    Signer 2 key
    H08mGNGN+NxX/snt+6eX2Q1HjjfDkOtotglshHi7xdsBdIrTVMCQbgQ5SdACNZ0B2AJcifK11nJj43SvaitSemI=

#### ROUND 2

  - Coordinator
      - `my_multisig_wallet.bsms`:

<!-- end list -->

    BSMS 1.0
    wsh(sortedmulti(1,[59865f44/48'/0'/0'/2']026d15412460ba0d881c21837bb999233896085a9ed4e5445bd637c10e579768ba,[b7044ca6/48'/0'/0'/2']030baf0497ab406ff50cb48b4013abac8a0338758d2fd54cd934927afa57cc2062))#rzx9dffd
    No path restrictions
    bc1quqy523xu3l8che3s8vja8n33qtg0uyugr9l5z092s3wa50p8t7rqy6zumf

### Mode: `NO_ENCRYPTION`

#### ROUND 1

  - Coordinator
      - M-of-N: 2/2
      - ADDRESS\_TYPE: NATIVE\_SEGWIT
      - TOKEN: 0x00

<!-- end list -->

  - Signer 1
      - MASTER\_KEY\_FINGERPRINT: 1cf0bf7e
      - PRIVATE\_KEY (m/48'/0'/0'/2'):
        L3q1sg7iso1L3QfzB1riC9bQpqMynWyBeuLLSKwCDGkHkahB7MgU
      - XPUB (m/48'/0'/0'/2'):
        xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz
      - Legacy signature
      - `signer_1_key.bsms`:

<!-- end list -->

    BSMS 1.0
    00
    [1cf0bf7e/48'/0'/0'/2']xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz
    Signer 1 key
    IB7v+qi1b+Xrwm/3bF+Rjl8QbIJ/FMQ40kUsOOQo1SqUWn5QlFWbBD8BKPRetfo1L1N7DmYjVscZNsmMrqRJGWw=

  - Signer 2
      - MASTER\_KEY\_FINGERPRINT: 4fc1dd4a
      - PRIVATE\_KEY (m/48'/0'/0'/2'):
        L4JNkJfLBDyWfTLbKJ1H3w56GUMsvdfjCkzRo5RHXfJ6bdHqm6cN
      - XPUB (m/48'/0'/0'/2'):
        xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj
      - Legacy signature
      - `signer_2_key.bsms`:

<!-- end list -->

    BSMS 1.0
    00
    [4fc1dd4a/48'/0'/0'/2']xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj
    Signer 2 key
    HzUa4Z76PFHMl54flIIF3XKiHZ+KbWjjxCEG5G3ZqZSqTd6OgTiFFLqq9PXJXdfYm6/cnL8IVWQgjFF9DQhIqQs=

#### ROUND 2

  - Coordinator
      - `my_multisig_wallet.bsms`:

<!-- end list -->

    BSMS 1.0
    wsh(sortedmulti(2,[1cf0bf7e/48'/0'/0'/2']xpub6FL8FhxNNUVnG64YurPd16AfGyvFLhh7S2uSsDqR3Qfcm6o9jtcMYwh6DvmcBF9qozxNQmTCVvWtxLpKTnhVLN3Pgnu2D3pAoXYFgVyd8Yz/**,[4fc1dd4a/48'/0'/0'/2']xpub6EebMbEps7ZcV3FYEnddRsvrFWDrt2tiPmCeM7pPXQEmphvq9ZfJ1LWFUDjf3vxCeBuPrfyGrMazWUsYsetrnHatQZVLJH7LsgCjtMqdzgj/**))
    /0/*,/1/*
    bc1qrgc6p3kylfztu06ysl752gwwuekhvtfh9vr7zg43jvu60mutamcsv948ej

### Mode: `STANDARD` Encryption

#### ROUND 1

  - Coordinator
      - M-of-N: 2/2
      - ADDRESS\_TYPE: NATIVE\_SEGWIT
      - TOKEN (hex): a54044308ceac9b7
          - TOKEN (decimal): 11907592390080907703
          - TOKEN (mnemonic): pipe acquire around border prosper swift
      - ENCRYPTION\_KEY (hex):
        7673ffd9efd70336a5442eda0b31457f7b6cdf7b42fe17f274434df55efa9839

<!-- end list -->

  - Signer 1
      - MASTER\_KEY\_FINGERPRINT: b7868815
      - PRIVATE\_KEY (m/48'/0'/0'/2'):
        KyKvR9kf8r7ZVtdn3kB9ifipr6UKnTNTpWJkGZbHwARDCz5iZ39E
      - XPUB (m/48'/0'/0'/2'):
        xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh
      - Legacy signature
      - `signer_1_key.bsms`:

<!-- end list -->

    BSMS 1.0
    a54044308ceac9b7
    [b7868815/48'/0'/0'/2']xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh
    Signer 1 key
    H8DYht5P6ko0bQqDV6MtUxpzBSK+aVHxbvMavA5byvLrOlCEGmO1WFR7k2wu42J6dxXD8vrmDQSnGq5MTMMbZ98=

  - Signer 1 encryption
      - HMAC\_KEY (hex):
        3d4c422806ba8964c9ee45070cd675c024d96648a0ddb4001325818c84951de2
      - MAC (hex):
        fbdbdb64e6a8231c342131d9f13dcd5a954b4c5021658fa5afcb3fc74dc82706
      - IV (hex) : fbdbdb64e6a8231c342131d9f13dcd5a
      - CIPHERTEXT (hex):
        53f491cfd1431c292d922ea5a5dec3eb8ddaa6ed38ae109e7b040f0f23013e89a89b4d27476761a01197a3277850b2bc1621ae626efe65f2081eec6eb571c4f787bf1c49d061b43f70fd73cb3f37fa591d2400973ac0644c8941a83f1d4155e98f01fa2fdeb9f86c2e2413154fd18566a28fb0d9d8bd6172efabcfa6dab09ee7029bf3dd43376df52c118a6d291ec168f4ec7f7df951dfc6135fd8cb4b234da62eaea6017dfe5ca418f083e02e3aba2962ba313ba17b6468c7672fb218329a9f3fe4e4887fb87dac57c63ebff0e715a44498d18de8afc10e1cfeb46a1fc65ce871fef8a43b289305433a90c342d025aa4c19454fcfbcf911e9e2f928d5affd0536a6ddc2e816
      - `signer_1_key.dat`:
            fbdbdb64e6a8231c342131d9f13dcd5a954b4c5021658fa5afcb3fc74dc8270653f491cfd1431c292d922ea5a5dec3eb8ddaa6ed38ae109e7b040f0f23013e89a89b4d27476761a01197a3277850b2bc1621ae626efe65f2081eec6eb571c4f787bf1c49d061b43f70fd73cb3f37fa591d2400973ac0644c8941a83f1d4155e98f01fa2fdeb9f86c2e2413154fd18566a28fb0d9d8bd6172efabcfa6dab09ee7029bf3dd43376df52c118a6d291ec168f4ec7f7df951dfc6135fd8cb4b234da62eaea6017dfe5ca418f083e02e3aba2962ba313ba17b6468c7672fb218329a9f3fe4e4887fb87dac57c63ebff0e715a44498d18de8afc10e1cfeb46a1fc65ce871fef8a43b289305433a90c342d025aa4c19454fcfbcf911e9e2f928d5affd0536a6ddc2e816

<!-- end list -->

  - Signer 2
      - MASTER\_KEY\_FINGERPRINT: eedff89a
      - PRIVATE\_KEY (m/48'/0'/0'/2'):
        Kz1ijnkDXmc65NWTYdg47DDaQgSGJAPfhJG9Unm36oqZPpPXuNR6
      - XPUB (m/48'/0'/0'/2'):
        xpub6EhJvMneoLWAf8cuyLBLQiKiwh89RAmqXEqYeFuaCEHdHwxSRfzLrUxKXEBap7nZSHAYP7Jfq6gZmucotNzpMQ9Sb1nTqerqW8hrtmx6Y6o
      - Legacy signature
      - `signer_2_key.bsms`:

<!-- end list -->

    BSMS 1.0
    a54044308ceac9b7
    [eedff89a/48'/0'/0'/2']xpub6EhJvMneoLWAf8cuyLBLQiKiwh89RAmqXEqYeFuaCEHdHwxSRfzLrUxKXEBap7nZSHAYP7Jfq6gZmucotNzpMQ9Sb1nTqerqW8hrtmx6Y6o
    Signer 2 key
    H/IHW5dMGYsrRdYEz3ux+kKnkWBtxHzfYkREpnYbco38VnMvIxCbDuf7iu6960qDhBLR/RLjlb9UPtLmCMbczDE=

  - Signer 2 encryption
      - HMAC\_KEY (hex):
        3d4c422806ba8964c9ee45070cd675c024d96648a0ddb4001325818c84951de2
      - MAC (hex):
        383d05b7351a2cef7cca2850450f5efbbc4a3f8ea35707dda87a3692f0f2ebae
      - IV (hex) : 383d05b7351a2cef7cca2850450f5efb
      - CIPHERTEXT (hex):
        71860b7c69f3a7665c3c3e85c45735bff78535a37ec6610b724627c73696820d519a9251703b17626b63898580233bebbb310aedbc370224b044ee19600bfe583445a6f26fb9bb5790bae516892655adb0e5dfc12be4609c2e0818d4f1f3bfccc4cd1a36f419d6cd842c913ae81eef4865ad473c32c3ee69cd98d6d0a088e2abdd01fe68b5c0503bb9183f9a912506204e5a9c6bd5a1626ff7eac30312a0b85004307c525e52fa3ad45a0b02eabc8cfaea0215bb6e60ee5f32d6673955290e008fbaef362977a21fd9830e3a604f9bb318cdcde456eae91dbedaa069bcd1efb0f981d5b0e502bd4dada903205458a00914887226a8dde317c02a8be4342acb97a8fee79fbe23
      - `signer_2_key.dat`:
            383d05b7351a2cef7cca2850450f5efbbc4a3f8ea35707dda87a3692f0f2ebae71860b7c69f3a7665c3c3e85c45735bff78535a37ec6610b724627c73696820d519a9251703b17626b63898580233bebbb310aedbc370224b044ee19600bfe583445a6f26fb9bb5790bae516892655adb0e5dfc12be4609c2e0818d4f1f3bfccc4cd1a36f419d6cd842c913ae81eef4865ad473c32c3ee69cd98d6d0a088e2abdd01fe68b5c0503bb9183f9a912506204e5a9c6bd5a1626ff7eac30312a0b85004307c525e52fa3ad45a0b02eabc8cfaea0215bb6e60ee5f32d6673955290e008fbaef362977a21fd9830e3a604f9bb318cdcde456eae91dbedaa069bcd1efb0f981d5b0e502bd4dada903205458a00914887226a8dde317c02a8be4342acb97a8fee79fbe23

#### ROUND 2

  - Coordinator
      - `my_multisig_wallet.bsms`:

<!-- end list -->

    BSMS 1.0
    wsh(sortedmulti(2,[b7868815/48'/0'/0'/2']xpub6FA5rfxJc94K1kNtxRby1hoHwi7YDyTWwx1KUR3FwskaF6HzCbZMz3zQwGnCqdiFeMTPV3YneTGS2YQPiuNYsSvtggWWMQpEJD4jXU7ZzEh/**,[eedff89a/48'/0'/0'/2']xpub6EhJvMneoLWAf8cuyLBLQiKiwh89RAmqXEqYeFuaCEHdHwxSRfzLrUxKXEBap7nZSHAYP7Jfq6gZmucotNzpMQ9Sb1nTqerqW8hrtmx6Y6o/**))
    /0/*,/1/*
    bc1qhs4u273g4azq7kqqpe6vh5wfhasfmrq7nheyzsnq77humd7rwtkqagvakf

  - Coordinator encryption
      - HMAC\_KEY (hex):
        3d4c422806ba8964c9ee45070cd675c024d96648a0ddb4001325818c84951de2
      - MAC (hex):
        734ce791b466861945e1ef6f74c63faec590793de54831f0036b28d08714b71a
      - IV (hex) : 734ce791b466861945e1ef6f74c63fae
      - CIPHERTEXT (hex):
        273cad18a5e1eff37dba6d850749594c9a3fd32b2069e8c69983ea269c5044b6bcaea26d9dbc8ad5d28bb8abfa02e3bfc7632fcc5c2b76e9abb1982ff11295858cfe44a8b97110ae970f58fff3fb6477f38ca9609eec78eedb1d640eaba489fd5e41e787b8d0bde48f1fa99cca641cabbee0f513fb1040cb73df10a57c9a34e4efcb069cd4c75467442c15d878ed9f40e3dffb98294931a6da4f444ae46f739b7fe002ce19fcfe71b05b9783d797ba45d568febbc8a2b0850da67f349d8567342352e1712c3d2a7ea1b2721df5efdb844431f0e5dcfa4acacb194c20785c9bb6dde90d64352fc913e9073b3b416be713bcc7632c821bbfddafa6199d471c54fb899f347f5fc706787ccaa82332dc8b93aeb3de3497d8e5c75f0f5d718c74bc6f8194fe999948e517f1c98398d9cb907d200f1d045394704b074dfb10e587f54fd78e95ef4bcbe77bf1376b390c3f47c91c12b2ed14073ea56bceab41f924302e62183c456b06d96b3da30439cb4320c764a0d6d1b3dabc06fc
      - `my_multisig_wallet.dat`:
            734ce791b466861945e1ef6f74c63faec590793de54831f0036b28d08714b71a273cad18a5e1eff37dba6d850749594c9a3fd32b2069e8c69983ea269c5044b6bcaea26d9dbc8ad5d28bb8abfa02e3bfc7632fcc5c2b76e9abb1982ff11295858cfe44a8b97110ae970f58fff3fb6477f38ca9609eec78eedb1d640eaba489fd5e41e787b8d0bde48f1fa99cca641cabbee0f513fb1040cb73df10a57c9a34e4efcb069cd4c75467442c15d878ed9f40e3dffb98294931a6da4f444ae46f739b7fe002ce19fcfe71b05b9783d797ba45d568febbc8a2b0850da67f349d8567342352e1712c3d2a7ea1b2721df5efdb844431f0e5dcfa4acacb194c20785c9bb6dde90d64352fc913e9073b3b416be713bcc7632c821bbfddafa6199d471c54fb899f347f5fc706787ccaa82332dc8b93aeb3de3497d8e5c75f0f5d718c74bc6f8194fe999948e517f1c98398d9cb907d200f1d045394704b074dfb10e587f54fd78e95ef4bcbe77bf1376b390c3f47c91c12b2ed14073ea56bceab41f924302e62183c456b06d96b3da30439cb4320c764a0d6d1b3dabc06fc

### Mode: `EXTENDED` Encryption

#### ROUND 1

  - Coordinator
      - M-of-N: 2/3
      - ADDRESS\_TYPE: NESTED\_SEGWIT
      - TOKEN for Signer 1 (hex): 108a2360adb302774eb521daebbeda5e
          - TOKEN (decimal): 21984902443033505423410071144203475550
          - ENCRYPTION\_KEY (hex):
            63dc1e57dfdc21fa11109d5088be01fb8078a383d2296925ad2b7612b7179777
      - TOKEN for Signer 2 (hex): d3fabc873b98165254fe18a71b5335b0
          - TOKEN (decimal): 281769005132501859744421970528095647152
          - ENCRYPTION\_KEY (hex):
            3dc860a53471ec03af14617fef60921cf215b45a9d684462fa65b9d804ad3ee7
      - TOKEN for Signer 3 (hex): 78a7d5e7549453d719150de5459c9ce5
          - TOKEN (decimal): 160378811550692397333855096016467696869
          - ENCRYPTION\_KEY (hex):
            62b90b4c08c03a0ee872e57aae73f9acfafb6cc09d20b5c9bc0bafaef33619db

<!-- end list -->

  - Signer 1
      - MASTER\_KEY\_FINGERPRINT: 793cc70b
      - PRIVATE\_KEY (m/48'/0'/0'/1'):
        L1ZEgZ4zNYxyNc8UyeqwyKW1UHVMp9sxwPgSi3s9SW8mc7KsiSwJ
      - XPUB (m/48'/0'/0'/1'):
        xpub6ErVmcYYHmavsMgxEcTZyzN5sqth1ZyRpFNJC26ij1wYGC2SBKYrgt9yariSbn7HLRoZUvhUhmPfsRTPrdhhGFscpPZzmch6UTdmRP1aZUj
      - Legacy signature
      - `signer_1_key.bsms`:

<!-- end list -->

    BSMS 1.0
    108a2360adb302774eb521daebbeda5e
    [793cc70b/48'/0'/0'/1']xpub6ErVmcYYHmavsMgxEcTZyzN5sqth1ZyRpFNJC26ij1wYGC2SBKYrgt9yariSbn7HLRoZUvhUhmPfsRTPrdhhGFscpPZzmch6UTdmRP1aZUj
    Signer 1 key
    ILG47LpCtjoD9UxL87jo5QFqA90t8g9fDQp/KBojdKgPPGB1pMx2bf9hPdORNZIOdCc/2+Gs6AOs3BEK9ubIuBw=

  - Signer 1 encryption
      - HMAC\_KEY (hex):
        1162cdace4ac9fcde1f96924b93714143d057a701de83ebaed248d1c9154f9fd
      - MAC (hex):
        ea12776c73de4bd5ea57c2d19eb8e0be856ac0d7f5651f7b74be4563d61ba5b1
      - IV (hex) : ea12776c73de4bd5ea57c2d19eb8e0be
      - CIPHERTEXT (hex):
        a36f34232bff47a853092654a718fea4f5f57d6a1f3d38fede04e2414da12c90cefc24ef662f736886d9a7fd6e7db636ca47217803c86b7fbcebe4ad6b71cffc261069c135bd2b2430fb2b446ff0203df34fbbc6801243e8a930b9d0cd3a9b160b8dcdc9131ce6e97641e6314b3285ff341013f302e308c1b2eba7ced0103a8999fe2bd86f844392938e7926cd26d023b764d0b8ff92b2fbdf995884c738414b83563ef2a0050279bf46d0e8271ea5d6af8154847c5736129a7a83a35a3cc747b2be4b389886cb57456678353b60473ebc4ab85d9c9131a17a1e288717343d9008825b16c48d7e93927f37b530033192c67b70dec0411a3e5952d2525c7eb80721676e1a6299248c17f8078202f3bb0932e9f263b0ab
      - `signer_1_key.dat`:
            ea12776c73de4bd5ea57c2d19eb8e0be856ac0d7f5651f7b74be4563d61ba5b1a36f34232bff47a853092654a718fea4f5f57d6a1f3d38fede04e2414da12c90cefc24ef662f736886d9a7fd6e7db636ca47217803c86b7fbcebe4ad6b71cffc261069c135bd2b2430fb2b446ff0203df34fbbc6801243e8a930b9d0cd3a9b160b8dcdc9131ce6e97641e6314b3285ff341013f302e308c1b2eba7ced0103a8999fe2bd86f844392938e7926cd26d023b764d0b8ff92b2fbdf995884c738414b83563ef2a0050279bf46d0e8271ea5d6af8154847c5736129a7a83a35a3cc747b2be4b389886cb57456678353b60473ebc4ab85d9c9131a17a1e288717343d9008825b16c48d7e93927f37b530033192c67b70dec0411a3e5952d2525c7eb80721676e1a6299248c17f8078202f3bb0932e9f263b0ab

<!-- end list -->

  - Signer 2
      - MASTER\_KEY\_FINGERPRINT: b3118e52
      - PRIVATE\_KEY (m/48'/0'/0'/1'):
        L4SnPjcHszMg3Wi2YYxEYnzM2zFeFkFr5NcLZ18YQeyJwaSFbTud
      - XPUB (m/48'/0'/0'/1'):
        xpub6Du5Jn6eYZE96ccmAc1ZTFPzdnzrvqfG4mpamDun2qZYKywoiQJMCbS3kWWMr6U3XW6s125RLsaPABWgv2yA749ieaMe67FxkTjMsbcxCch
      - Legacy signature
      - `signer_2_key.bsms`:

<!-- end list -->

    BSMS 1.0
    d3fabc873b98165254fe18a71b5335b0
    [b3118e52/48'/0'/0'/1']xpub6Du5Jn6eYZE96ccmAc1ZTFPzdnzrvqfG4mpamDun2qZYKywoiQJMCbS3kWWMr6U3XW6s125RLsaPABWgv2yA749ieaMe67FxkTjMsbcxCch
    Signer 2 key
    IDK4d/oO0pgfrwRu4Zb8vqlPEmJb9aKT1K2CCnI3RKepVAKs3fZsBrypcCdQfUy1TG/3O5vAR3gjldxcCA1Wzg8=

  - Signer 2 encryption
      - HMAC\_KEY (hex):
        43a4e704bd1bade703023004b00290f1a7b005474a581d869a217068eedf3f57
      - MAC (hex):
        4a3ff970d027010e83b4fbf2845a23907a301b3df692a9265e2ca679697ac718
      - IV (hex) : 4a3ff970d027010e83b4fbf2845a2390
      - CIPHERTEXT (hex):
        c8f4a6a6714eff90aa48cbefe6750c2ee3cc72182eb455e964f0ba59ada3ecd758c29f0fab7e33aaa82a340a18d9c793ddab09dc7e714864faac1ecea370d4f102533b739da38aa0491433f35eadec08f203685f04d1f6ec35d397d99e4f8096a5691075e3f54fd9ff58faf947f276bbe1031f827b274bd2f60fcb526add7058889104b189d7da22ac7be1f7ddd380bbebd5c6983a8a3c5fa86913e3d23c40935072ce03d9bdeb07791dc836d44b4d4c62f364d0e4f3580369ea8f6ebb774b7fda4a7ac6f5ae6b2f52b10cd71bdf3cdb5889e77d5eb1f2f647b798cdd6b3e5b964c9265daea3668d7e4cb53f724151923da1a87bbcd2abd8b164de474d865c51af69885431d26f88a5c8eea7d0dfdb52ca622017808a
      - `signer_2_key.dat`:
            4a3ff970d027010e83b4fbf2845a23907a301b3df692a9265e2ca679697ac718c8f4a6a6714eff90aa48cbefe6750c2ee3cc72182eb455e964f0ba59ada3ecd758c29f0fab7e33aaa82a340a18d9c793ddab09dc7e714864faac1ecea370d4f102533b739da38aa0491433f35eadec08f203685f04d1f6ec35d397d99e4f8096a5691075e3f54fd9ff58faf947f276bbe1031f827b274bd2f60fcb526add7058889104b189d7da22ac7be1f7ddd380bbebd5c6983a8a3c5fa86913e3d23c40935072ce03d9bdeb07791dc836d44b4d4c62f364d0e4f3580369ea8f6ebb774b7fda4a7ac6f5ae6b2f52b10cd71bdf3cdb5889e77d5eb1f2f647b798cdd6b3e5b964c9265daea3668d7e4cb53f724151923da1a87bbcd2abd8b164de474d865c51af69885431d26f88a5c8eea7d0dfdb52ca622017808a

<!-- end list -->

  - Signer 3
      - MASTER\_KEY\_FINGERPRINT: 842bd2ed
      - PRIVATE\_KEY (m/48'/0'/0'/1'):
        L1ehZHpo2UFHc1yaBWDU4bKVycUwcU2TESm92wbfq6xK6qpZZJP6
      - XPUB (m/48'/0'/0'/1'):
        xpub6Ex81KopPkEt9hJiWHabYy8LNsSR4A7sUQoFBk9dR8XxHrr4p9HrYWN3NCf5uwfopHnQkCG7FYnZMztKbtRtbh6tzZC4xtHPbmVVxRSN7ic
      - Legacy signature
      - `signer_3_key.bsms`:

<!-- end list -->

    BSMS 1.0
    78a7d5e7549453d719150de5459c9ce5
    [842bd2ed/48'/0'/0'/1']xpub6Ex81KopPkEt9hJiWHabYy8LNsSR4A7sUQoFBk9dR8XxHrr4p9HrYWN3NCf5uwfopHnQkCG7FYnZMztKbtRtbh6tzZC4xtHPbmVVxRSN7ic
    Signer 3 key
    IL77mML0xo/O9dJn0T5EpQLuyRPPrdpgVJbtsdAugW5iX0MQ3Ci0f8jVnXu68Xm07CYjYGKX8af72jmkQKhNud0=

  - Signer 3 encryption
      - HMAC\_KEY (hex):
        ab93ce7bf0f91c62a66d00ea9bf5e5c00b854ee2cfc2fb06f6eeff738abcdc26
      - MAC (hex):
        e82cfcccbd4bd4d3b76e28133eecd13f7362f4a8b4c4baa3e5f6ba2dfb4d69b8
      - IV (hex) : e82cfcccbd4bd4d3b76e28133eecd13f
      - CIPHERTEXT (hex):
        b44433f0b564ec35a1e71371f25844088084b47402c90d52fee7237167b58a60a28c234af9123e104773136e8446d799541c8566882787caee7cd1fa8628aba63aa9e9d7cca0ddee92f96dd881535b19a131a1f487a1909e42d62945fd0ba08dacd7dc09a22ffe47e0410b8b85df92e4a8bbf9b46f0062da02e3ae94144a00bae917acc1246d8d1a4dca105708f55379caefef9d4c152f56b65ab4bd7b48f60233f57ba6d705387c79aeaa2a279e3314004bf16fcd7e7d2adef34b0ab3c22bc5461f2c09dce69065605e4fb96958c55984391712b3547e3914ad4ecca2c088be280dfcfe374a112515674aeca57b885e81dbef6a353ca387f4514db3158eb69f0d2725d42ad8102c05c26ad501d48b889c624035ead4
      - `signer_3_key.dat`:
            e82cfcccbd4bd4d3b76e28133eecd13f7362f4a8b4c4baa3e5f6ba2dfb4d69b8b44433f0b564ec35a1e71371f25844088084b47402c90d52fee7237167b58a60a28c234af9123e104773136e8446d799541c8566882787caee7cd1fa8628aba63aa9e9d7cca0ddee92f96dd881535b19a131a1f487a1909e42d62945fd0ba08dacd7dc09a22ffe47e0410b8b85df92e4a8bbf9b46f0062da02e3ae94144a00bae917acc1246d8d1a4dca105708f55379caefef9d4c152f56b65ab4bd7b48f60233f57ba6d705387c79aeaa2a279e3314004bf16fcd7e7d2adef34b0ab3c22bc5461f2c09dce69065605e4fb96958c55984391712b3547e3914ad4ecca2c088be280dfcfe374a112515674aeca57b885e81dbef6a353ca387f4514db3158eb69f0d2725d42ad8102c05c26ad501d48b889c624035ead4

#### ROUND 2

  - Coordinator
      - `my_multisig_wallet.bsms`:

<!-- end list -->

    BSMS 1.0
    sh(wsh(multi(2,[793cc70b/48'/0'/0'/1']xpub6ErVmcYYHmavsMgxEcTZyzN5sqth1ZyRpFNJC26ij1wYGC2SBKYrgt9yariSbn7HLRoZUvhUhmPfsRTPrdhhGFscpPZzmch6UTdmRP1aZUj/**,[b3118e52/48'/0'/0'/1']xpub6Du5Jn6eYZE96ccmAc1ZTFPzdnzrvqfG4mpamDun2qZYKywoiQJMCbS3kWWMr6U3XW6s125RLsaPABWgv2yA749ieaMe67FxkTjMsbcxCch/**,[842bd2ed/48'/0'/0'/1']xpub6Ex81KopPkEt9hJiWHabYy8LNsSR4A7sUQoFBk9dR8XxHrr4p9HrYWN3NCf5uwfopHnQkCG7FYnZMztKbtRtbh6tzZC4xtHPbmVVxRSN7ic/**)))
    /0/*,/1/*
    3GzMtFXahiu4TpGNGFc4bHMvAcvz5vVQrT

  - Send to Signer 1:
      - HMAC\_KEY (hex):
        1162cdace4ac9fcde1f96924b93714143d057a701de83ebaed248d1c9154f9fd
      - MAC (hex):
        01bf557b6d44b3fbf07f8ec155cbdec42d85d856e174342563dd83b40ad7c025
      - IV (hex) : 01bf557b6d44b3fbf07f8ec155cbdec4
      - CIPHERTEXT (hex):
        617ed25b4b8fd88b806cbebcc1731b071465514a805f7ba2de60e291bc9493f31aa0f9b0665ba822cf9a2e21c02649b5c3f7dbad317ae898292cb6fe992520f68c0ebe9d1434b348af10453f1be0a392a616d43ba21e5e7fa3c995dce54db947fe5dbad4a9a77f37b3aef58c54ee3e496c8312d3033359aed0de8cf28b82035ee7a38c9b23c9d95682fb15936bf2247546d2ba9b3ada605f5c89f0a3bbaa86cb4b5dded9a65004912c0afbbfd01f0115447f5625e8523f9de16165d32c4b21103d8ac965e2f7e17641ee1a8c5902e8dbb461c6c7d05141f7bba66b8b3608037fb251b55fa461c9441c6427921545a34a1798127d5bf9cc92423f7e62c769e232c65db8cc5124577012d49941143c3b4758212a8afa0475c9b3597da2e99d585039339b7d73611aa277878d212875051683053db9c630391e0b32356523e9fa8a58a334e16fe6650472f336ddaa8c587992b6c0c0e480b680261579a11cf9d036614abc113dde53653273f5ce82ea0bc10e38ca52ac66838aa49ff46c3a7d5096db439c15d3c2e8de55e4ac7315a57eb9997f219c378af86c858867ce583ed84e4d9c68aecfbca9ebff16b0ac91531125e273b215db688ffe52c8033eb78914b87c0fa2001c52e90c92765712e50384ddcf4d0953ac3cc8137abcb2a85d603a6cc207472677
      - `my_multisig_wallet_for_signer_1.dat`:
            01bf557b6d44b3fbf07f8ec155cbdec42d85d856e174342563dd83b40ad7c025617ed25b4b8fd88b806cbebcc1731b071465514a805f7ba2de60e291bc9493f31aa0f9b0665ba822cf9a2e21c02649b5c3f7dbad317ae898292cb6fe992520f68c0ebe9d1434b348af10453f1be0a392a616d43ba21e5e7fa3c995dce54db947fe5dbad4a9a77f37b3aef58c54ee3e496c8312d3033359aed0de8cf28b82035ee7a38c9b23c9d95682fb15936bf2247546d2ba9b3ada605f5c89f0a3bbaa86cb4b5dded9a65004912c0afbbfd01f0115447f5625e8523f9de16165d32c4b21103d8ac965e2f7e17641ee1a8c5902e8dbb461c6c7d05141f7bba66b8b3608037fb251b55fa461c9441c6427921545a34a1798127d5bf9cc92423f7e62c769e232c65db8cc5124577012d49941143c3b4758212a8afa0475c9b3597da2e99d585039339b7d73611aa277878d212875051683053db9c630391e0b32356523e9fa8a58a334e16fe6650472f336ddaa8c587992b6c0c0e480b680261579a11cf9d036614abc113dde53653273f5ce82ea0bc10e38ca52ac66838aa49ff46c3a7d5096db439c15d3c2e8de55e4ac7315a57eb9997f219c378af86c858867ce583ed84e4d9c68aecfbca9ebff16b0ac91531125e273b215db688ffe52c8033eb78914b87c0fa2001c52e90c92765712e50384ddcf4d0953ac3cc8137abcb2a85d603a6cc207472677

<!-- end list -->

  - Send to Signer 2:
      - HMAC\_KEY (hex):
        43a4e704bd1bade703023004b00290f1a7b005474a581d869a217068eedf3f57
      - MAC (hex):
        974ba77900c43c463dadaa6eaf24aaeb1b25b443cf155229b719bcbf8b343092
      - IV (hex) : 974ba77900c43c463dadaa6eaf24aaeb
      - CIPHERTEXT (hex):
        86288c97a6341974a35015f97fbbc8db7655639c839fc438706f82fce36a82dd17e2d4d4a674516c4fc5c3a33d6097dd8fc5c6605018946741ed9f58d8fe530a808f16f0dd705cacfd273e34a158bd7566774dd31506b8280e448fabb72d0e7dfc05cee1142b61921dfaf0b0039d885cc0aa76c429025efc2ba49a8af15b58e75a5a83ba4838a9a4c9f13725f5aecefd8511513d93797f37b93150b9dca725685883188e39142dd8d3cf4b617c7936bdb3875415bbf6dfb2fe1a39ae2aed9fd2909aebd0355a5cc9a55bcb84048c851a1873948e495180f336edeb63f54bcf83feaa4d2453251260e24293e49815c2369c1c045083c412c973987fd7c9296a71cda424823ed32380ba442394500b7d2d2335818099090aaf08ca4e180869c546f58da4cb4ff0f95b796a35c40ea455e2ddd3e08bc494ffddc706aaf4d479f4f359e6a89a90df7c9b8f23cab355855a72b90795a0db83a96bce0dd4f77e3f58c0957b4ffe9663251565098e6c31fd4bbf3e1295faaff05e29912d9c37cb944da379a9b2193b466910d05a681e53a2dbe5aa18a2b4874153fe36d8a1aa4cc6e612bd6dbc9abb8e1e61b927fc5458d8e1be9536cd462e4c37672af7928c39e94bdc124a2da7b1bd3cad2cfe559adc33e62eb45bff89db8a47a72dda4f49f21c01a9432f4802a1
      - `my_multisig_wallet_for_signer_2.dat`:
            974ba77900c43c463dadaa6eaf24aaeb1b25b443cf155229b719bcbf8b34309286288c97a6341974a35015f97fbbc8db7655639c839fc438706f82fce36a82dd17e2d4d4a674516c4fc5c3a33d6097dd8fc5c6605018946741ed9f58d8fe530a808f16f0dd705cacfd273e34a158bd7566774dd31506b8280e448fabb72d0e7dfc05cee1142b61921dfaf0b0039d885cc0aa76c429025efc2ba49a8af15b58e75a5a83ba4838a9a4c9f13725f5aecefd8511513d93797f37b93150b9dca725685883188e39142dd8d3cf4b617c7936bdb3875415bbf6dfb2fe1a39ae2aed9fd2909aebd0355a5cc9a55bcb84048c851a1873948e495180f336edeb63f54bcf83feaa4d2453251260e24293e49815c2369c1c045083c412c973987fd7c9296a71cda424823ed32380ba442394500b7d2d2335818099090aaf08ca4e180869c546f58da4cb4ff0f95b796a35c40ea455e2ddd3e08bc494ffddc706aaf4d479f4f359e6a89a90df7c9b8f23cab355855a72b90795a0db83a96bce0dd4f77e3f58c0957b4ffe9663251565098e6c31fd4bbf3e1295faaff05e29912d9c37cb944da379a9b2193b466910d05a681e53a2dbe5aa18a2b4874153fe36d8a1aa4cc6e612bd6dbc9abb8e1e61b927fc5458d8e1be9536cd462e4c37672af7928c39e94bdc124a2da7b1bd3cad2cfe559adc33e62eb45bff89db8a47a72dda4f49f21c01a9432f4802a1

<!-- end list -->

  - Send to Signer 3:
      - HMAC\_KEY (hex):
        ab93ce7bf0f91c62a66d00ea9bf5e5c00b854ee2cfc2fb06f6eeff738abcdc26
      - MAC (hex):
        bb3c93b67d758f244de7ee73e5e61261cea6dff5b3852df8faf265cdf1c73dae
      - IV (hex) : bb3c93b67d758f244de7ee73e5e61261
      - CIPHERTEXT (hex):
        7ac33bd9719a3cef6c68e09b3c9677565418933f188bbe50dc70f46329706732fe28ab230468e2a8798d3fbf641867d5b3322113204a372e7650ed06cf94d6df5cc7425b1b3a07690a32e12fd9cdad2c9f42d496c1b02215a7d8d63565aa4935bb2b087af39eebc02d4a2d30a4dbf1e72b9a0dab11473c7254ecf9065eb4f9d80a164c489d5fdae0d15d97b6100b79c3999b91341dfb4f599f738d4d631ae413c17b55daa09a67cb34b40d89c26f0e95ddfbf416033f869da32e502815d720bb342ec1c0e5c6910c598f32162016229cd37ea030b4d3b60f560105abb75531dc960ddf6830c26604c67c2da05b8adc45297dda58b2da4671104969b819cdf1c362bc20d7bdfe4a2fbdb79b4a69e285434d991c269e3d23ce3d95675a0acbec2cae04a310581148d3422c1c0a621fb6d79ecac1743b0e76837389b67cd4734ec5ab560c43a183de35fa98834e1f347a0c0c9b14273b76233f55f04553efcde873de92d766f3cdc5e56bc649bf0cc4951f051619ee9b931cd3872044b0e62ea2c2dacad978dbb8df3afa0b9386535278c295c6a30a56950e57f805770568e937ffafbadb226120991d5ec10effa9f4334800010d141a2ddddc00ac743efa821af37f69840487e4db48036c1e0730788cddbca2f68b3769ec6989d76161e6605af50651b6e86e
      - `my_multisig_wallet_for_signer_3.dat`:
            bb3c93b67d758f244de7ee73e5e61261cea6dff5b3852df8faf265cdf1c73dae7ac33bd9719a3cef6c68e09b3c9677565418933f188bbe50dc70f46329706732fe28ab230468e2a8798d3fbf641867d5b3322113204a372e7650ed06cf94d6df5cc7425b1b3a07690a32e12fd9cdad2c9f42d496c1b02215a7d8d63565aa4935bb2b087af39eebc02d4a2d30a4dbf1e72b9a0dab11473c7254ecf9065eb4f9d80a164c489d5fdae0d15d97b6100b79c3999b91341dfb4f599f738d4d631ae413c17b55daa09a67cb34b40d89c26f0e95ddfbf416033f869da32e502815d720bb342ec1c0e5c6910c598f32162016229cd37ea030b4d3b60f560105abb75531dc960ddf6830c26604c67c2da05b8adc45297dda58b2da4671104969b819cdf1c362bc20d7bdfe4a2fbdb79b4a69e285434d991c269e3d23ce3d95675a0acbec2cae04a310581148d3422c1c0a621fb6d79ecac1743b0e76837389b67cd4734ec5ab560c43a183de35fa98834e1f347a0c0c9b14273b76233f55f04553efcde873de92d766f3cdc5e56bc649bf0cc4951f051619ee9b931cd3872044b0e62ea2c2dacad978dbb8df3afa0b9386535278c295c6a30a56950e57f805770568e937ffafbadb226120991d5ec10effa9f4334800010d141a2ddddc00ac743efa821af37f69840487e4db48036c1e0730788cddbca2f68b3769ec6989d76161e6605af50651b6e86e

## Acknowledgement

Special thanks to Pavol Rusnak, Dmitry Petukhov, Christopher Allen,
Craig Raw, Robert Spigler, Gregory Sanders, Ta Tat Tai, Michael Flaxman,
Pieter Wuille, Salvatore Ingala, Andrew Chow and others for their
feedback on the specification.

## References

Related mailing list threads:

  - <https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2021-February/018385.html>
  - <https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2021-April/018732.html>
