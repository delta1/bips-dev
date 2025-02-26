+++
title = "Schnorr Signatures for secp256k1"
date = 2020-01-19
weight = 340
in_search_index = true

[taxonomies]
authors = ["Pieter Wuille", "Jonas Nick", "Tim Ruffing"]
status = ["Draft"]

[extra]
bip = 340
status = ["Draft"]
github = "https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki"
+++

``` 
  BIP: 340
  Title: Schnorr Signatures for secp256k1
  Author: Pieter Wuille <pieter.wuille@gmail.com>
          Jonas Nick <jonasd.nick@gmail.com>
          Tim Ruffing <crypto@timruffing.de>
  Comments-Summary: No comments yet.
  Comments-URI: https://github.com/bitcoin/bips/wiki/Comments:BIP-0340
  Status: Draft
  Type: Standards Track
  License: BSD-2-Clause
  Created: 2020-01-19
  Post-History: 2018-07-06: https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-July/016203.html [bitcoin-dev] Schnorr signatures BIP
```

## Introduction

### Abstract

This document proposes a standard for 64-byte Schnorr signatures over
the elliptic curve *secp256k1*.

### Copyright

This document is licensed under the 2-clause BSD license.

### Motivation

Bitcoin has traditionally used
[ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm)
signatures over the [secp256k1 curve](https://www.secg.org/sec2-v2.pdf)
with [SHA256](https://en.wikipedia.org/wiki/SHA-2) hashes for
authenticating transactions. These are
[standardized](https://www.secg.org/sec1-v2.pdf), but have a number of
downsides compared to [Schnorr
signatures](http://publikationen.ub.uni-frankfurt.de/opus4/files/4280/schnorr.pdf)
over the same curve:

  - **Provable security**: Schnorr signatures are provably secure. In
    more detail, they are *strongly unforgeable under chosen message
    attack (SUF-CMA)*\[1\] [in the random oracle model assuming the
    hardness of the elliptic curve discrete logarithm problem
    (ECDLP)](https://www.di.ens.fr/~pointche/Documents/Papers/2000_joc.pdf)
    and [in the generic group model assuming variants of preimage and
    second preimage resistance of the used hash
    function](http://www.neven.org/papers/schnorr.pdf)\[2\]. In
    contrast, the [best known results for the provable security of
    ECDSA](https://nbn-resolving.de/urn:nbn:de:hbz:294-60803) rely on
    stronger assumptions.
  - **Non-malleability**: The SUF-CMA security of Schnorr signatures
    implies that they are non-malleable. On the other hand, ECDSA
    signatures are inherently malleable\[3\]; a third party without
    access to the secret key can alter an existing valid signature for a
    given public key and message into another signature that is valid
    for the same key and message. This issue is discussed in
    [BIP62](bip-0062.mediawiki "wikilink") and
    [BIP146](bip-0146.mediawiki "wikilink").
  - **Linearity**: Schnorr signatures provide a simple and efficient
    method that enables multiple collaborating parties to produce a
    signature that is valid for the sum of their public keys. This is
    the building block for various higher-level constructions that
    improve efficiency and privacy, such as multisignatures and others
    (see Applications below).

For all these advantages, there are virtually no disadvantages, apart
from not being standardized. This document seeks to change that. As we
propose a new standard, a number of improvements not specific to Schnorr
signatures can be made:

  - **Signature encoding**: Instead of using
    [DER](https://en.wikipedia.org/wiki/X.690#DER_encoding)-encoding for
    signatures (which are variable size, and up to 72 bytes), we can use
    a simple fixed 64-byte format.
  - **Public key encoding**: Instead of using
    [*compressed*](https://www.secg.org/sec1-v2.pdf) 33-byte encodings
    of elliptic curve points which are common in Bitcoin today, public
    keys in this proposal are encoded as 32 bytes.
  - **Batch verification**: The specific formulation of ECDSA signatures
    that is standardized cannot be verified more efficiently in batch
    compared to individually, unless additional witness data is added.
    Changing the signature scheme offers an opportunity to address this.
  - **Completely specified**: To be safe for usage in consensus systems,
    the verification algorithm must be completely specified at the byte
    level. This guarantees that nobody can construct a signature that is
    valid to some verifiers but not all. This is traditionally not a
    requirement for digital signature schemes, and the lack of exact
    specification for the DER parsing of ECDSA signatures has caused
    problems for Bitcoin [in the
    past](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2015-July/009697.html),
    needing [BIP66](bip-0066.mediawiki "wikilink") to address it. In
    this document we aim to meet this property by design. For batch
    verification, which is inherently non-deterministic as the verifier
    can choose their batches, this property implies that the outcome of
    verification may only differ from individual verifications with
    negligible probability, even to an attacker who intentionally tries
    to make batch- and non-batch verification differ.

By reusing the same curve and hash function as Bitcoin uses for ECDSA,
we are able to retain existing mechanisms for choosing secret and public
keys, and we avoid introducing new assumptions about the security of
elliptic curves and hash functions.

## Description

We first build up the algebraic formulation of the signature scheme by
going through the design choices. Afterwards, we specify the exact
encodings and operations.

### Design

**Schnorr signature variant** Elliptic Curve Schnorr signatures for
message *m* and public key *P* generally involve a point *R*, integers
*e* and *s* picked by the signer, and the base point *G* which satisfy
*e = hash(R || m)* and *s⋅G = R + e⋅P*. Two formulations exist,
depending on whether the signer reveals *e* or *R*:

1.  Signatures are pairs *(e, s)* that satisfy *e = hash(s⋅G - e⋅P ||
    m)*. This variant avoids minor complexity introduced by the encoding
    of the point *R* in the signature (see paragraphs "Encoding R and
    public key point P" and "Implicit Y coordinates" further below in
    this subsection). Moreover, revealing *e* instead of *R* allows for
    potentially shorter signatures: Whereas an encoding of *R*
    inherently needs about 32 bytes, the hash *e* can be tuned to be
    shorter than 32 bytes, and [a short hash of only 16 bytes suffices
    to provide SUF-CMA security at the target security level of 128
    bits](http://www.neven.org/papers/schnorr.pdf). However, a major
    drawback of this optimization is that finding collisions in a short
    hash function is easy. This complicates the implementation of secure
    signing protocols in scenarios in which a group of mutually
    distrusting signers work together to produce a single joint
    signature (see Applications below). In these scenarios, which are
    not captured by the SUF-CMA model due its assumption of a single
    honest signer, a promising attack strategy for malicious co-signers
    is to find a collision in the hash function in order to obtain a
    valid signature on a message that an honest co-signer did not intend
    to sign.
2.  Signatures are pairs *(R, s)* that satisfy *s⋅G = R + hash(R ||
    m)⋅P*. This supports batch verification, as there are no elliptic
    curve operations inside the hashes. Batch verification enables
    significant speedups.\[4\]

Since we would like to avoid the fragility that comes with short hashes,
the *e* variant does not provide significant advantages. We choose the
*R*-option, which supports batch verification.

**Key prefixing** Using the verification rule above directly makes
Schnorr signatures vulnerable to "related-key attacks" in which a third
party can convert a signature *(R, s)* for public key *P* into a
signature *(R, s + a⋅hash(R || m))* for public key *P + a⋅G* and the
same message *m*, for any given additive tweak *a* to the signing key.
This would render signatures insecure when keys are generated using
[BIP32's unhardened
derivation](bip-0032.mediawiki#public-parent-key--public-child-key "wikilink")
and other methods that rely on additive tweaks to existing keys such as
Taproot.

To protect against these attacks, we choose *key prefixed*\[5\] Schnorr
signatures which means that the public key is prefixed to the message in
the challenge hash input. This changes the equation to *s⋅G = R + hash(R
|| P || m)⋅P*. [It can be shown](https://eprint.iacr.org/2015/1135.pdf)
that key prefixing protects against related-key attacks with additive
tweaks. In general, key prefixing increases robustness in multi-user
settings, e.g., it seems to be a requirement for proving the MuSig
multisignature scheme secure (see Applications below).

We note that key prefixing is not strictly necessary for transaction
signatures as used in Bitcoin currently, because signed transactions
indirectly commit to the public keys already, i.e., *m* contains a
commitment to *pk*. However, this indirect commitment should not be
relied upon because it may change with proposals such as
SIGHASH\_NOINPUT ([BIP118](bip-0118.mediawiki "wikilink")), and would
render the signature scheme unsuitable for other purposes than signing
transactions, e.g., [signing ordinary
messages](https://bitcoin.org/en/developer-reference#signmessage).

**Encoding R and public key point P** There exist several possibilities
for encoding elliptic curve points:

1.  Encoding the full X and Y coordinates of *P* and *R*, resulting in a
    64-byte public key and a 96-byte signature.
2.  Encoding the full X coordinate and one bit of the Y coordinate to
    determine one of the two possible Y coordinates. This would result
    in 33-byte public keys and 65-byte signatures.
3.  Encoding only the X coordinate, resulting in 32-byte public keys and
    64-byte signatures.

Using the first option would be slightly more efficient for verification
(around 10%), but we prioritize compactness, and therefore choose option
3.

**Implicit Y coordinates** In order to support efficient verification
and batch verification, the Y coordinate of *P* and of *R* cannot be
ambiguous (every valid X coordinate has two possible Y coordinates). We
have a choice between several options for symmetry breaking:

1.  Implicitly choosing the Y coordinate that is in the lower half.
2.  Implicitly choosing the Y coordinate that is even\[6\].
3.  Implicitly choosing the Y coordinate that is a quadratic residue
    (i.e. has a square root modulo *p*).

The second option offers the greatest compatibility with existing key
generation systems, where the standard 33-byte compressed public key
format consists of a byte indicating the oddness of the Y coordinate,
plus the full X coordinate. To avoid gratuitous incompatibilities, we
pick that option for *P*, and thus our X-only public keys become
equivalent to a compressed public key that is the X-only key prefixed by
the byte 0x02. For consistency, the same is done for *R*\[7\].

Despite halving the size of the set of valid public keys, implicit Y
coordinates are not a reduction in security. Informally, if a fast
algorithm existed to compute the discrete logarithm of an X-only public
key, then it could also be used to compute the discrete logarithm of a
full public key: apply it to the X coordinate, and then optionally
negate the result. This shows that breaking an X-only public key can be
at most a small constant term faster than breaking a full one.\[8\].

**Tagged Hashes** Cryptographic hash functions are used for multiple
purposes in the specification below and in Bitcoin in general. To make
sure hashes used in one context can't be reinterpreted in another one,
hash functions can be tweaked with a context-dependent tag name, in such
a way that collisions across contexts can be assumed to be infeasible.
Such collisions obviously can not be ruled out completely, but only for
schemes using tagging with a unique name. As for other schemes
collisions are at least less likely with tagging than without.

For example, without tagged hashing a BIP340 signature could also be
valid for a signature scheme where the only difference is that the
arguments to the hash function are reordered. Worse, if the BIP340 nonce
derivation function was copied or independently created, then the nonce
could be accidentally reused in the other scheme leaking the secret key.

This proposal suggests to include the tag by prefixing the hashed data
with *SHA256(tag) || SHA256(tag)*. Because this is a 64-byte long
context-specific constant and the *SHA256* block size is also 64 bytes,
optimized implementations are possible (identical to SHA256 itself, but
with a modified initial state). Using SHA256 of the tag name itself is
reasonably simple and efficient for implementations that don't choose to
use the optimization.

**Final scheme** As a result, our final scheme ends up using public key
*pk* which is the X coordinate of a point *P* on the curve whose Y
coordinate is even and signatures *(r,s)* where *r* is the X coordinate
of a point *R* whose Y coordinate is even. The signature satisfies *s⋅G
= R + tagged\_hash(r || pk || m)⋅P*.

### Specification

The following conventions are used, with constants as defined for
[secp256k1](https://www.secg.org/sec2-v2.pdf). We note that adapting
this specification to other elliptic curves is not straightforward and
can result in an insecure scheme\[9\].

  - Lowercase variables represent integers or byte arrays.
      - The constant *p* refers to the field size,
        *0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F*.
      - The constant *n* refers to the curve order,
        *0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141*.
  - Uppercase variables refer to points on the curve with equation
    *y<sup>2</sup> = x<sup>3</sup> + 7* over the integers modulo *p*.
      - *is\_infinite(P)* returns whether or not *P* is the point at
        infinity.
      - *x(P)* and *y(P)* are integers in the range *0..p-1* and refer
        to the X and Y coordinates of a point *P* (assuming it is not
        infinity).
      - The constant *G* refers to the base point, for which *x(G) =
        0x79BE667EF9DCBBAC55A06295CE870B07029BFCDB2DCE28D959F2815B16F81798*
        and *y(G) =
        0x483ADA7726A3C4655DA4FBFC0E1108A8FD17B448A68554199C47D08FFB10D4B8*.
      - Addition of points refers to the usual [elliptic curve group
        operation](https://en.wikipedia.org/wiki/Elliptic_curve#The_group_law).
      - [Multiplication (⋅) of an integer and a
        point](https://en.wikipedia.org/wiki/Elliptic_curve_point_multiplication)
        refers to the repeated application of the group operation.
  - Functions and operations:
      - *||* refers to byte array concatenation.
      - The function *x\[i:j\]*, where *x* is a byte array and *i, j ≥
        0*, returns a *(j - i)*-byte array with a copy of the *i*-th
        byte (inclusive) to the *j*-th byte (exclusive) of *x*.
      - The function *bytes(x)*, where *x* is an integer, returns the
        32-byte encoding of *x*, most significant byte first.
      - The function *bytes(P)*, where *P* is a point, returns
        *bytes(x(P))*.
      - The function *int(x)*, where *x* is a 32-byte array, returns the
        256-bit unsigned integer whose most significant byte first
        encoding is *x*.
      - The function *has\_even\_y(P)*, where *P* is a point for which
        *not is\_infinite(P)*, returns *y(P) mod 2 = 0*.
      - The function *lift\_x(x)*, where *x* is a 256-bit unsigned
        integer, returns the point *P* for which *x(P) = x*<ref>

`   Given a candidate X coordinate `*`x`*` in the range `*`0..p-1`*`, there exist either exactly two or exactly zero valid Y coordinates. If no valid Y coordinate exists, then `*`x`*` is not a valid X coordinate either, i.e., no point `*`P`*` exists for which `*`x(P)`` 
 ``=`` 
 ``x`*`. The valid Y coordinates for a given candidate `*`x`*` are the square roots of `*`c`` 
 ``=``   ``x`<sup>`3`</sup>`   ``+``   ``7``   ``mod`` 
 ``p`*` and they can be computed as `*`y``   ``=`` 
 ``±c`<sup>`(p+1)/4`</sup>`   ``mod``   ``p`*` (see `[`Quadratic`` 
 ``residue`](https://en.wikipedia.org/wiki/Quadratic_residue#Prime_or_prime_power_modulus)`) if they exist, which can be checked by squaring and comparing with `*`c`*`.`</ref>` and `*`has_even_y(P)`*`, or fails if `*`x`*` is greater than `*`p-1`*` or no such point exists. The function `*`lift_x(x)`*` is equivalent to the following pseudocode:`

  -   - Fail if *x ≥ p*.
      - Let *c = x<sup>3</sup> + 7 mod p*.
      - Let *y = c<sup>(p+1)/4</sup> mod p*.
      - Fail if *c ≠ y<sup>2</sup> mod p*.
      - Return the unique point *P* such that *x(P) = x* and *y(P) = y*
        if *y mod 2 = 0* or *y(P) = p-y* otherwise.

  -   - The function *hash<sub>tag</sub>(x)* where *tag* is a UTF-8
        encoded tag name and *x* is a byte array returns the 32-byte
        hash *SHA256(SHA256(tag) || SHA256(tag) || x)*.

#### Public Key Generation

Input:

  - The secret key *sk*: a 32-byte array, freshly generated uniformly at
    random

The algorithm *PubKey(sk)* is defined as:

  - Let *d' = int(sk)*.
  - Fail if *d' = 0* or *d' ≥ n*.
  - Return *bytes(d'⋅G)*.

Note that we use a very different public key format (32 bytes) than the
ones used by existing systems (which typically use elliptic curve points
as public keys, or 33-byte or 65-byte encodings of them). A side effect
is that *PubKey(sk) = PubKey(bytes(n - int(sk))*, so every public key
has two corresponding secret keys.

#### Public Key Conversion

As an alternative to generating keys randomly, it is also possible and
safe to repurpose existing key generation algorithms for ECDSA in a
compatible way. The secret keys constructed by such an algorithm can be
used as *sk* directly. The public keys constructed by such an algorithm
(assuming they use the 33-byte compressed encoding) need to be converted
by dropping the first byte. Specifically,
[BIP32](bip-0032.mediawiki "wikilink") and schemes built on top of it
remain usable.

#### Default Signing

Input:

  - The secret key *sk*: a 32-byte array
  - The message *m*: a 32-byte array
  - Auxiliary random data *a*: a 32-byte array

The algorithm *Sign(sk, m)* is defined as:

  - Let *d' = int(sk)*
  - Fail if *d' = 0* or *d' ≥ n*
  - Let *P = d'⋅G*
  - Let ''d = d' '' if *has\_even\_y(P)*, otherwise let ''d = n - d' ''.
  - Let *t* be the byte-wise xor of *bytes(d)* and
    *hash<sub>BIP0340/aux</sub>(a)*\[10\].
  - Let *rand = hash<sub>BIP0340/nonce</sub>(t || bytes(P) || m)*\[11\].
  - Let *k' = int(rand) mod n*\[12\].
  - Fail if *k' = 0*.
  - Let *R = k'⋅G*.
  - Let ''k = k' '' if *has\_even\_y(R)*, otherwise let ''k = n - k' ''.
  - Let *e = int(hash<sub>BIP0340/challenge</sub>(bytes(R) || bytes(P)
    || m)) mod n*.
  - Let *sig = bytes(R) || bytes((k + ed) mod n)*.
  - If *Verify(bytes(P), m, sig)* (see below) returns failure,
    abort\[13\].
  - Return the signature *sig*.

The auxiliary random data should be set to fresh randomness generated at
signing time, resulting in what is called a *synthetic nonce*. Using 32
bytes of randomness is optimal. If obtaining randomness is expensive, 16
random bytes can be padded with 16 null bytes to obtain a 32-byte array.
If randomness is not available at all at signing time, a simple counter
wide enough to not repeat in practice (e.g., 64 bits or wider) and
padded with null bytes to a 32 byte-array can be used, or even the
constant array with 32 null bytes. Using any non-repeating value
increases protection against [fault injection
attacks](https://moderncrypto.org/mail-archive/curves/2017/000925.html).
Using unpredictable randomness additionally increases protection against
other side-channel attacks, and is **recommended whenever available**.
Note that while this means the resulting nonce is not deterministic, the
randomness is only supplemental to security. The normal security
properties (excluding side-channel attacks) do not depend on the quality
of the signing-time RNG.

#### Alternative Signing

It should be noted that various alternative signing algorithms can be
used to produce equally valid signatures. The 32-byte *rand* value may
be generated in other ways, producing a different but still valid
signature (in other words, this is not a *unique* signature scheme).
**No matter which method is used to generate the *rand* value, the value
must be a fresh uniformly random 32-byte string which is not even
partially predictable for the attacker.** For nonces without randomness
this implies that the same inputs must not be presented in another
context. This can be most reliably accomplished by not reusing the same
private key across different signing schemes. For example, if the *rand*
value was computed as per RFC6979 and the same secret key is used in
deterministic ECDSA with RFC6979, the signatures can leak the secret key
through nonce reuse.

**Nonce exfiltration protection** It is possible to strengthen the nonce
generation algorithm using a second device. In this case, the second
device contributes randomness which the actual signer provably
incorporates into its nonce. This prevents certain attacks where the
signer device is compromised and intentionally tries to leak the secret
key through its nonce selection.

**Multisignatures** This signature scheme is compatible with various
types of multisignature and threshold schemes such as
[MuSig](https://eprint.iacr.org/2018/068), where a single public key
requires holders of multiple secret keys to participate in signing (see
Applications below). **It is important to note that multisignature
signing schemes in general are insecure with the *rand* generation from
the default signing algorithm above (or any other deterministic
method).**

**Precomputed public key data** For many uses the compressed 33-byte
encoding of the public key corresponding to the secret key may already
be known, making it easy to evaluate *has\_even\_y(P)* and *bytes(P)*.
As such, having signers supply this directly may be more efficient than
recalculating the public key from the secret key. However, if this
optimization is used and additionally the signature verification at the
end of the signing algorithm is dropped for increased efficiency,
signers must ensure the public key is correctly calculated and not taken
from untrusted sources.

#### Verification

Input:

  - The public key *pk*: a 32-byte array
  - The message *m*: a 32-byte array
  - A signature *sig*: a 64-byte array

The algorithm *Verify(pk, m, sig)* is defined as:

  - Let *P = lift\_x(int(pk))*; fail if that fails.
  - Let *r = int(sig\[0:32\])*; fail if *r ≥ p*.
  - Let *s = int(sig\[32:64\])*; fail if *s ≥ n*.
  - Let *e = int(hash<sub>BIP0340/challenge</sub>(bytes(r) || bytes(P)
    || m)) mod n*.
  - Let *R = s⋅G - e⋅P*.
  - Fail if *is\_infinite(R)*.
  - Fail if *not has\_even\_y(R)*.
  - Fail if *x(R) ≠ r*.
  - Return success iff no failure occurred before reaching this point.

For every valid secret key *sk* and message *m*,
*Verify(PubKey(sk),m,Sign(sk,m))* will succeed.

Note that the correctness of verification relies on the fact that
*lift\_x* always returns a point with an even Y coordinate. A
hypothetical verification algorithm that treats points as public keys,
and takes the point *P* directly as input would fail any time a point
with odd Y is used. While it is possible to correct for this by negating
points with odd Y coordinate before further processing, this would
result in a scheme where every (message, signature) pair is valid for
two public keys (a type of malleability that exists for ECDSA as well,
but we don't wish to retain). We avoid these problems by treating just
the X coordinate as public key.

#### Batch Verification

Input:

  - The number *u* of signatures
  - The public keys *pk<sub>1..u</sub>*: *u* 32-byte arrays
  - The messages *m<sub>1..u</sub>*: *u* 32-byte arrays
  - The signatures *sig<sub>1..u</sub>*: *u* 64-byte arrays

The algorithm *BatchVerify(pk<sub>1..u</sub>, m<sub>1..u</sub>,
sig<sub>1..u</sub>)* is defined as:

  - Generate *u-1* random integers *a<sub>2...u</sub>* in the range
    *1...n-1*. They are generated deterministically using a
    [CSPRNG](https://en.wikipedia.org/wiki/Cryptographically_secure_pseudorandom_number_generator)
    seeded by a cryptographic hash of all inputs of the algorithm, i.e.
    *seed = seed\_hash(pk<sub>1</sub>..pk<sub>u</sub> ||
    m<sub>1</sub>..m<sub>u</sub> || sig<sub>1</sub>..sig<sub>u</sub> )*.
    A safe choice is to instantiate *seed\_hash* with SHA256 and use
    [ChaCha20](https://tools.ietf.org/html/rfc8439) with key *seed* as a
    CSPRNG to generate 256-bit integers, skipping integers not in the
    range *1...n-1*.
  - For *i = 1 .. u*:
      - Let *P<sub>i</sub> = lift\_x(int(pk<sub>i</sub>))*; fail if it
        fails.
      - Let *r<sub>i</sub> = int(sig<sub>i</sub>\[0:32\])*; fail if
        *r<sub>i</sub> ≥ p*.
      - Let *s<sub>i</sub> = int(sig<sub>i</sub>\[32:64\])*; fail if
        *s<sub>i</sub> ≥ n*.
      - Let *e<sub>i</sub> =
        int(hash<sub>BIP0340/challenge</sub>(bytes(r<sub>i</sub>) ||
        bytes(P<sub>i</sub>) || m<sub>i</sub>)) mod n*.
      - Let *R<sub>i</sub> = lift\_x(r<sub>i</sub>)*; fail if
        *lift\_x(r<sub>i</sub>)* fails.
  - Fail if *(s<sub>1</sub> + a<sub>2</sub>s<sub>2</sub> + ... +
    a<sub>u</sub>s<sub>u</sub>)⋅G ≠ R<sub>1</sub> +
    a<sub>2</sub>⋅R<sub>2</sub> + ... + a<sub>u</sub>⋅R<sub>u</sub> +
    e<sub>1</sub>⋅P<sub>1</sub> +
    (a<sub>2</sub>e<sub>2</sub>)⋅P<sub>2</sub> + ... +
    (a<sub>u</sub>e<sub>u</sub>)⋅P<sub>u</sub>*.
  - Return success iff no failure occurred before reaching this point.

If all individual signatures are valid (i.e., *Verify* would return
success for them), *BatchVerify* will always return success. If at least
one signature is invalid, *BatchVerify* will return success with at most
a negligible probability.

## Applications

There are several interesting applications beyond simple signatures.
While recent academic papers claim that they are also possible with
ECDSA, consensus support for Schnorr signature verification would
significantly simplify the constructions.

### Multisignatures and Threshold Signatures

By means of an interactive scheme such as
[MuSig](https://eprint.iacr.org/2018/068), participants can aggregate
their public keys into a single public key which they can jointly sign
for. This allows *n*-of-*n* multisignatures which, from a verifier's
perspective, are no different from ordinary signatures, giving improved
privacy and efficiency versus *CHECKMULTISIG* or other means.

Moreover, Schnorr signatures are compatible with [distributed key
generation](https://web.archive.org/web/20031003232851/http://www.research.ibm.com/security/dkg.ps),
which enables interactive threshold signatures schemes, e.g., the
schemes described by [Stinson and Strobl
(2001)](http://cacr.uwaterloo.ca/techreports/2001/corr2001-13.ps) or
[Gennaro, Jarecki and Krawczyk
(2003)](https://web.archive.org/web/20060911151529/http://theory.lcs.mit.edu/~stasio/Papers/gjkr03.pdf).
These protocols make it possible to realize *k*-of-*n* threshold
signatures, which ensure that any subset of size *k* of the set of *n*
signers can sign but no subset of size less than *k* can produce a valid
Schnorr signature. However, the practicality of the existing schemes is
limited: most schemes in the literature have been proven secure only for
the case *k-1 \< n/2*, are not secure when used concurrently in multiple
sessions, or require a reliable broadcast mechanism to be secure.
Further research is necessary to improve this situation.

### Adaptor Signatures

[Adaptor
signatures](https://download.wpsoftware.net/bitcoin/wizardry/mw-slides/2018-05-18-l2/slides.pdf)
can be produced by a signer by offsetting his public nonce *R* with a
known point *T = t⋅G*, but not offsetting the signature's *s* value. A
correct signature (or partial signature, as individual signers'
contributions to a multisignature are called) on the same message with
same nonce will then be equal to the adaptor signature offset by *t*,
meaning that learning *t* is equivalent to learning a correct signature.
This can be used to enable atomic swaps or even [general payment
channels](https://eprint.iacr.org/2018/472) in which the atomicity of
disjoint transactions is ensured using the signatures themselves, rather
than Bitcoin script support. The resulting transactions will appear to
verifiers to be no different from ordinary single-signer transactions,
except perhaps for the inclusion of locktime refund logic.

Adaptor signatures, beyond the efficiency and privacy benefits of
encoding script semantics into constant-sized signatures, have
additional benefits over traditional hash-based payment channels.
Specifically, the secret values *t* may be reblinded between hops,
allowing long chains of transactions to be made atomic while even the
participants cannot identify which transactions are part of the chain.
Also, because the secret values are chosen at signing time, rather than
key generation time, existing outputs may be repurposed for different
applications without recourse to the blockchain, even multiple times.

### Blind Signatures

A blind signature protocol is an interactive protocol that enables a
signer to sign a message at the behest of another party without learning
any information about the signed message or the signature. Schnorr
signatures admit a very [simple blind signature
scheme](http://publikationen.ub.uni-frankfurt.de/files/4292/schnorr.blind_sigs_attack.2001.pdf)
which is however insecure because it's vulnerable to [Wagner's
attack](https://www.iacr.org/archive/crypto2002/24420288/24420288.pdf).
A known mitigation is to let the signer abort a signing session with a
certain probability, and the resulting scheme can be [proven secure
under non-standard cryptographic
assumptions](https://eprint.iacr.org/2019/877).

Blind Schnorr signatures could for example be used in [Partially Blind
Atomic
Swaps](https://github.com/ElementsProject/scriptless-scripts/blob/master/md/partially-blind-swap.md),
a construction to enable transferring of coins, mediated by an untrusted
escrow agent, without connecting the transactors in the public
blockchain transaction graph.

## Test Vectors and Reference Code

For development and testing purposes, we provide a [collection of test
vectors in CSV format](bip-0340/test-vectors.csv "wikilink") and a
naive, highly inefficient, and non-constant time [pure Python 3.7
reference implementation of the signing and verification
algorithm](bip-0340/reference.py "wikilink"). The reference
implementation is for demonstration purposes only and not to be used in
production environments.

## Changelog

To help implementors understand updates to this BIP, we keep a list of
substantial changes.

  - 2022-08: Fix function signature of lift\_x in reference code

## Footnotes

<references />

## Acknowledgements

This document is the result of many discussions around Schnorr based
signatures over the years, and had input from Johnson Lau, Greg Maxwell,
Andrew Poelstra, Rusty Russell, and Anthony Towns. The authors further
wish to thank all those who provided valuable feedback and reviews,
including the participants of the [structured
reviews](https://github.com/ajtowns/taproot-review).

1.  Informally, this means that without knowledge of the secret key but
    given valid signatures of arbitrary messages, it is not possible to
    come up with further valid signatures.
2.  A detailed security proof in the random oracle model, which
    essentially restates [the original security proof by Pointcheval and
    Stern](https://www.di.ens.fr/~pointche/Documents/Papers/2000_joc.pdf)
    more explicitly, can be found in [a paper by Kiltz, Masny and
    Pan](https://eprint.iacr.org/2016/191). All these security proofs
    assume a variant of Schnorr signatures that use *(e,s)* instead of
    *(R,s)* (see Design above). Since we use a unique encoding of *R*,
    there is an efficiently computable bijection that maps *(R,s)* to
    *(e,s)*, which allows to convert a successful SUF-CMA attacker for
    the *(e,s)* variant to a successful SUF-CMA attacker for the *(R,s)*
    variant (and vice-versa). Furthermore, the proofs consider a variant
    of Schnorr signatures without key prefixing (see Design above), but
    it can be verified that the proofs are also correct for the variant
    with key prefixing. As a result, all the aforementioned security
    proofs apply to the variant of Schnorr signatures proposed in this
    document.
3.  If *(r,s)* is a valid ECDSA signature for a given message and key,
    then *(r,n-s)* is also valid for the same message and key. If ECDSA
    is restricted to only permit one of the two variants (as Bitcoin
    does through a policy rule on the network), it can be
    [proven](https://nbn-resolving.de/urn:nbn:de:hbz:294-60803)
    non-malleable under stronger than usual assumptions.
4.  The speedup that results from batch verification can be demonstrated
    with the cryptography library
    [libsecp256k1](https://github.com/jonasnick/secp256k1/blob/schnorrsig-batch-verify/doc/speedup-batch.md).
5.  A limitation of committing to the public key (rather than to a short
    hash of it, or not at all) is that it removes the ability for public
    key recovery or verifying signatures against a short public key
    hash. These constructions are generally incompatible with batch
    verification.
6.  Since *p* is odd, negation modulo *p* will map even numbers to odd
    numbers and the other way around. This means that for a valid X
    coordinate, one of the corresponding Y coordinates will be even, and
    the other will be odd.
7.  An earlier version of this draft used the third option instead,
    based on a belief that this would in general trade signing
    efficiency for verification efficiency. When using Jacobian
    coordinates, a common optimization in ECC implementations, it is
    possible to determine if a Y coordinate is a quadratic residue by
    computing the Legendre symbol, without converting to affine
    coordinates first (which needs a modular inversion). As modular
    inverses and Legendre symbols have similar
    [performance](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-August/018081.html)
    in practice, this trade-off is not worth it.
8.  This can be formalized by a simple reduction that reduces an attack
    on Schnorr signatures with implicit Y coordinates to an attack to
    Schnorr signatures with explicit Y coordinates. The reduction works
    by reencoding public keys and negating the result of the hash
    function, which is modeled as random oracle, whenever the challenge
    public key has an explicit Y coordinate that is odd. A proof sketch
    can be found
    [here](https://medium.com/blockstream/reducing-bitcoin-transaction-sizes-with-x-only-pubkeys-f86476af05d7).
9.  Among other pitfalls, using the specification with a curve whose
    order is not close to the size of the range of the nonce derivation
    function is insecure.
10. The auxiliary random data is hashed (with a unique tag) as a
    precaution against situations where the randomness may be correlated
    with the private key itself. It is xored with the private key
    (rather than combined with it in a hash) to reduce the number of
    operations exposed to the actual secret key.
11. Including the [public key as input to the nonce
    hash](https://moderncrypto.org/mail-archive/curves/2020/001012.html)
    helps ensure the robustness of the signing algorithm by preventing
    leakage of the secret key if the calculation of the public key *P*
    is performed incorrectly or maliciously, for example if it is left
    to the caller for performance reasons.
12. Note that in general, taking a uniformly random 256-bit integer
    modulo the curve order will produce an unacceptably biased result.
    However, for the secp256k1 curve, the order is sufficiently close to
    *2<sup>256</sup>* that this bias is not observable (*1 - n /
    2<sup>256</sup>* is around *1.27 \* 2<sup>-128</sup>*).
13. Verifying the signature before leaving the signer prevents random or
    attacker provoked computation errors. This prevents publishing
    invalid signatures which may leak information about the secret key.
    It is recommended, but can be omitted if the computation cost is
    prohibitive.
