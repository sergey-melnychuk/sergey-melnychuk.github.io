---
layout: post
title:  "Learning ECDSA with arkworks-rs"
date:   2023-11-22 20:20:42 +0200
categories: rust crypto arkworks ecdsa bitcoin
---
While I was [doing-some-blockchain](https://github.com/sergey-melnychuk/doing-some-blockchain) recently, I have found out that Elliptic Curve algebra does not seem to fit easily into `u128`, so this is a second attepmt, now done properly, using [arkworks-rs](https://github.com/arkworks-rs).

TL/DR: [code](https://github.com/sergey-melnychuk/doing-some-curves).

Note: this is a purely educational example, meant just to show how abstract math can be implemented in real world using propert tools & libraries. Do not use this implementation for anything beyound education. Consider yourself warned.

For a warmup let's implement [DHKE](https://en.wikipedia.org/wiki/Diffie–Hellman_key_exchange) on a `secp256k1` elliptic curve (the one used in Bitcoin):

```rust
// cargo add rand ark-std ark-ff ark-secp256k1
#[test]
fn test_dhke() {
    use ark_std::{rand, UniformRand};
    use ark_secp256k1::{Affine, Fr as F, G_GENERATOR_X, G_GENERATOR_Y};

    let mut rng = rand::thread_rng();
    let g = Affine::new(G_GENERATOR_X, G_GENERATOR_Y);

    let a = F::rand(&mut rng);
    let ga = g * a;

    let b = F::rand(&mut rng);
    let gb = g * b;

    let one = ga + gb;
    let two = gb + ga;

    // shared secrets must match
    assert_eq!(one, two);
}
```

```
$ cargo test
...
running 1 test
test tests::test_dhke ... ok
```

Now [ECDSA](https://en.wikipedia.org/wiki/Elliptic_Curve_Digital_Signature_Algorithm): first add necessary dependencies:

```
cargo add sha2 hex
cargo add ark-serialize --features derive
```

So that final dependencies list looks like this:

```
[dependencies]
ark-ff = "0.4.2"
ark-secp256k1 = "0.4.0"
ark-serialize = { version = "0.4.2", features = ["derive"] }
ark-std = "0.4.0"
hex = "0.4.3"
rand = "0.8.5"
sha2 = "0.10.8"
```

First step is to derive a Public Key (PK) from a Secret Key (SK) - with SK being a scalar (just a regular number, not yet a point on the curve), it is enough to just multiply curve's generator point (G) by the scalar to get a point on the curve. Thanks to the [Discrete Logarithm problem](https://en.wikipedia.org/wiki/Discrete_logarithm), inversing this operation (and thus "cracking" a secret key from a public one) is considered extremely computationally intense (for the numbers of a certain size, say 256 bits).

For simplisity the API I'm presenting is going to use raw bytes slices and vectors to represent both public and secret keys.

```rust
pub fn get_pk(sk: &[u8]) -> Vec<u8> {
    // Derive Public Key from a Secret Key:
    // 
    // SK - secret key (scalar)
    // G - generator point
    // 
    // PK = G * SK

    let g = Affine::new(G_GENERATOR_X, G_GENERATOR_Y);
    let sk = F::from_be_bytes_mod_order(&sk);
    let pk = g * sk;

    into_bytes(&pk)
}
```

The signing part, pretty straightforward math:

```rust
pub fn sig(sk: &[u8], msg: &[u8]) -> (Vec<u8>, Vec<u8>) {
    // Signing:
    // 
    // SK - secret key
    // PK - public key (PK = SK * G)
    // m - message
    // H - sha256
    // (r, s) - signature
    // 
    // k = H(H(SK) || H(m))
    // R = k * G
    // r = R.x
    // s = k' * (h + r * SK)

    let k = hash(&[&hash(&[sk]), &hash(&[msg])]);
    let k = F::from_be_bytes_mod_order(&k);
    let ki = k.inverse().unwrap();

    let h = hash(&[msg]);
    let h = F::from_be_bytes_mod_order(&h);

    let sk = F::from_be_bytes_mod_order(&sk);

    let g = Affine::new(G_GENERATOR_X, G_GENERATOR_Y);
    let r = g * k;
    let r: Projective = from_bytes(&into_bytes(&r));
    let rx = F::from(r.x.into_bigint());

    let s = ki * (h + rx * sk);

    (into_bytes(&rx), into_bytes(&s))
}
```

The signature verification part: the whole "trick" is that `(h + r * SK)` and it's modular inverse cancel each other out, so the original `R` is restored and compared to provided with the signature one (x coordinates really).

```rust
pub fn ver(pk: &[u8], msg: &[u8], sig: (&[u8], &[u8])) -> bool {
    // Verification
    // 
    // R = (h * s') * G + (r * s') * PK
    // For a valid signature: R.x == sig.r
    //
    // R = (h * s') * G + (r * s') * PK
    // R = (h * s') * G + (r * s') * SK * G
    // R = (h + r * SK) * s' * G
    // R = (h + r * SK) * (k' * (h + r * SK))' * G
    // R = (h + r * SK) * k * (h + r * SK)' * G
    // R = k * G

    let pk: Projective = from_bytes(&pk);

    let (rx, s) = sig;
    let rx: types::Number = from_bytes(rx);
    let s: types::Number = from_bytes(s);
    let si = s.inverse().unwrap();

    let h = hash(&[msg]);
    let h = F::from_be_bytes_mod_order(&h);

    let g = Affine::new(G_GENERATOR_X, G_GENERATOR_Y);
    let r = g * h * si + pk * rx * si;
    let r: Projective = from_bytes(&into_bytes(&r));
    let x1 = F::from(r.x.into_bigint());

    into_bytes(&rx) == into_bytes(&x1)
}
```

Does it work? Checking:

```rust
#[test]
fn test_ecdsa() {
    use super::*;

    let sk = "43cdf7c47a34cac01e717ad098bde292c2b3972719da38b7d38706be25706d4f";
    let sk = hex::decode(sk).unwrap();
    let pk = get_pk(&sk);

    let msg = b"the quick brown fox jumps over the lazy dog";
    let (r, s) = sig(&sk, msg);
    assert!(ver(&pk, msg, (&r, &s)));
}
```

Seems like it does:

```
$ cargo test
...
running 2 tests
test tests::test_dhke ... ok
test tests::test_ecdsa ... ok
...
```

To test against random secret keys:

```rust
#[test]
fn test_ecdsa() {
    use super::*;
    
    use ark_std::{rand, UniformRand};
    let mut rng = rand::thread_rng();
    let a = F::rand(&mut rng);
    let sk = into_bytes(&a);

    // let sk = "43cdf7c47a34cac01e717ad098bde292c2b3972719da38b7d38706be25706d4f";
    // let sk = hex::decode(sk).unwrap();
    let pk = get_pk(&sk);

    let msg = b"the quick brown fox jumps over the lazy dog";
    let (r, s) = sig(&sk, msg);
    assert!(ver(&pk, msg, (&r, &s)));
}
```

Full code is on [GitHub](https://github.com/sergey-melnychuk/doing-some-curves). Implementations of `{from, into}_bytes` and `hash` are pretty straightforward, so not providing them here for brevity. They are present in the repository of course.

ONE NOTE THOUGH:

There is one tricky moment in this implementation. Attentive reader might have noticed weird "flipping" (serializing to bytes and then deserializing back to point) of `R` happening in both signing and verification parts (`let r: Projective = from_bytes(&into_bytes(&r));`). The problem I had without such "flip", was that the original point `R` was restored (byte representation did match the original one), but `x` coordinate did not match the original `R`'s `x` coordinate, for reasons I don't quite understand. Pretty sure I must have misused `arkworks-rs`, and there is a proper way to express it, but I don't know it yet! Leaving this as an excercise for a reader :) As long as tests pass, I think the educational purposes of this example implementation are achieved.

References:
- [Getting started with arkworks-rs](https://medium.com/@prajwolgyawali/getting-started-with-arkworks-rs-e5ceaca895a9)
- [Applied Elliptic Curve Cryptography](https://ventral.digital/posts/2023/8/22/applied-elliptic-curve-cryptography)
