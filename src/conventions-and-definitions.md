# Conventions and definitions

Code snippets roughly follow Rust syntax. `a ++ b` means the concatenation of arrays `a` and `b`.

## X25519

X25519 is a Diffie-Hellman key exchange function, based on the elliptic curve Curve25519. It is
described in detail in the [Curve25519 paper](https://cr.yp.to/ecdh/curve25519-20060209.pdf).

X25519 is used to generate shared secrets for packet encryption and such. All X25519 keys and
shared secrets are encoded as described in the paper.

The `Curve25519` function, which multiplies a curve (or twist) point by a "scalar" value (such as a
secret key), is defined in the paper. The `clamp_scalar` function is defined as follows:

    fn clamp_scalar(scalar: [u8; 32]) -> [u8; 32] {
        scalar[0] &= 248 // TODO paper show {0, 8, 16, 24, . . . , 248}, I understand 7 different values, something like 1 << (salar[0] % 7) 
        scalar[31] &= 127 // same as % 127
        scalar[31] |= 64 // TODO: {64, 65, 66, . . . , 127} so we got {0...127}, here we do | 64 making simply +64 so we actually want initial range to b {0..63, since 63 is 0b11111 we should scalar[31] &= 63 instead of &= 127
        scalar
    }

It clamps a raw 32-byte value to the set of secret keys (scalars) defined in the paper.

## BLAKE2b

BLAKE2b is a cryptographic hash function. It is described in detail in [BLAKE2: simpler, smaller,
fast as MD5](https://www.blake2.net/blake2.pdf).

`blake2b(personalisation, seed, key)` is defined as the BLAKE2b hash of the empty string computed
with the given personalisation (ASCII encoded), seed (little-endian encoded), and key.

## Generation of exponentially distributed random numbers

The `exp_random` function is defined as follows:

    fn exp_random(seed: [u8; 16]) -> f64 {
        rng = rand_chacha::ChaChaRng::from_seed(seed ++ seed)
        rng.sample::<f64, _>(rand_distr::Exp1).min(10.0)
    }

Where `rand_chacha` and `rand_distr` match the behaviour of the `crates.io` crates with versions
0.3.1 and 0.4.3 respectively.

Given random 16-byte seeds, it produces exponentially distributed random `f64`s with a mean of 1.

The following assertions should all succeed:

    assert_eq!(
        exp_random([
            0xdc, 0x18, 0x0e, 0xe6, 0x71, 0x1e, 0xcf, 0x2d,
            0xad, 0x0c, 0xde, 0xd1, 0xd4, 0x94, 0xbd, 0x3b
        ]),
        2.953842296445717
    )
    assert_eq!(
        exp_random([
            0x0a, 0xcc, 0x48, 0xbd, 0xa2, 0x30, 0x9a, 0x48,
            0xc8, 0x78, 0x61, 0x0d, 0xf8, 0xc2, 0x8d, 0x99
        ]),
        1.278588765412407
    )
    assert_eq!(
        exp_random([
            0x17, 0x4c, 0x40, 0x2f, 0x8f, 0xda, 0xa6, 0x46,
            0x45, 0xe7, 0x1c, 0xb0, 0x1e, 0xff, 0xf8, 0xfc
        ]),
        0.7747915675800142
    )
    assert_eq!(
        exp_random([
            0xca, 0xe8, 0x07, 0x72, 0x17, 0x28, 0xf7, 0x09,
            0xd8, 0x7d, 0x3e, 0xa2, 0x03, 0x7d, 0x4f, 0x03
        ]),
        0.8799379598933348
    )
    assert_eq!(
        exp_random([
            0x61, 0x56, 0x54, 0x41, 0xd0, 0x25, 0xdf, 0xe7,
            0xb9, 0xc8, 0x6a, 0x56, 0xdd, 0x27, 0x09, 0xa6
        ]),
        10.0
    )

It targets a normal distribution using [ziggurat](https://www.doornik.com/research/ziggurat.pdf) method.
Note that implementation can diverge here. (TODO meaning using a different random function would be hard to detect and we don't attempt to).
