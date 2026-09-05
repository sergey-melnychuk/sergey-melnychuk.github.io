---
layout: post
title: "Closing the Book: Pairings, BLS and CM Curves in Rust"
date: 2026-05-27 10:00:00 +0100
categories: elliptic-curves cryptography rust bilinear-pairing pairings bls
---

TL/DR: a follow-up to [Elliptic Curves math in Rust]({{ site.url }}{% post_url 2026-02-11-eec-rust %}). The [`ecc`](https://github.com/sergey-melnychuk/ecc) repo now covers **every chapter** of Michael Rosing's *Elliptic Curve Cryptography for Developers* — chapter 13 (elliptic curves over extension fields), 15 (Miller's algorithm, Weil and Tate pairings), 16–17 (tiny-field pairing demos), 18 (BLS signatures with aggregation), plus the two "tooling" chapters 6 (base-curve parameters) and 14 (pairing-friendly curve search + Hilbert class polynomials). BLS verifies end-to-end on the book's real-size `curve_11` parameter file, and the full curve-generation pipeline can reproduce `curve_11` from its `(p, t)` alone — its CM discriminant turns out to be the Heegner number `D = -163`. **82 tests pass.**

### What was missing

After the [first post]({{ site.url }}{% post_url 2026-02-11-eec-rust %}), the repo had the core arithmetic spine (chapters 2, 3, 7–12) and both endpoints — DHKE/ECDSA (4–5) and the full Groth16-style SNARK (19). The middle was thin: pairings only existed inside the `snark/` module as a self-contained port, and the educational chapters that *use* pairings outside SNARKs (BLS signatures, the Weil/Tate demos on toy curves) were absent.

This update fills that gap and goes further — closing the loop on curve generation too:

| Ch | Topic | Rust |
|----|-------|------|
| 6  | base-curve params (160/256/384/512) + `.dat` loader | [`src/elliptic.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/elliptic.rs) |
| 13 | curves over GF(p^k) | [`src/poly_elliptic.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/poly_elliptic.rs) |
| 14 | pairing-friendly curve search + CM construction | [`src/curve_search.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/curve_search.rs), [`src/cm_curve.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/cm_curve.rs) |
| 15 | Weil/Tate via Miller | [`src/pairing.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/pairing.rs) |
| 16 | Weil tiny-curve demo | [`src/bin/weil.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/bin/weil.rs) |
| 17 | Tate tiny-curve demo | [`src/bin/tate.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/bin/tate.rs) |
| 18 | BLS signatures (tiny + real curve) | [`src/bls.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/bls.rs) + [`src/bin/bls.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/bin/bls.rs) |

### Chapter 13: elliptic curves over GF(p^k)

The same `y^2 = x^3 + a_4 x + a_6` equation, but now the coefficients of both the curve and every point are polynomials reduced by an irreducible. The Rust version reuses [`Polynomial`](https://github.com/sergey-melnychuk/ecc/blob/main/src/polynomial.rs) from chapter 8 and threads the irreducible and base modulus through explicitly:

```rust
pub struct PolyCurve {
    pub a4: Polynomial,
    pub a6: Polynomial,
    pub irrd: Polynomial,
    pub p: Modulus,
}

impl PolyCurve {
    pub fn add(&self, p: &PolyPoint, q: &PolyPoint) -> PolyPoint {
        if p.is_inf() { return q.clone(); }
        if q.is_inf() { return p.clone(); }

        let y_sum = p.y.add(&q.y);
        let (num, den) = if coef_zero_mod(&y_sum, &self.p) {
            let x_diff = poly_sub_mod(&p.x, &q.x, &self.p);
            if coef_zero_mod(&x_diff, &self.p) {
                return PolyPoint::inf();      // P + (-P) = O
            }
            // x1 ≠ x2 but y1 = -y2 — fall back to chord slope.
            (poly_sub_mod(&q.y, &p.y, &self.p),
             poly_sub_mod(&q.x, &p.x, &self.p))
        } else {
            // Unified slope: λ = (x1² + x1·x2 + x2² + a4) / (y1 + y2)
            let t1 = p.x.mul(&p.x, &self.irrd, &self.p);
            let t2 = p.x.mul(&q.x, &self.irrd, &self.p);
            let t3 = q.x.mul(&q.x, &self.irrd, &self.p);
            (t1.add(&t2).add(&t3).add(&self.a4), y_sum)
        };
        // ...
    }
}
```

Subtle thing the C code handles correctly that the Rust port needs too: the unified slope `λ = (Σxᵢ² + a₄) / (y₁+y₂)` breaks when `y₁ + y₂ = 0` *but* `x₁ ≠ x₂` — a chord whose endpoints happen to be additive inverses of each other in the y coordinate, without being negatives of each other as points. The fallback is the plain chord slope `(y₁-y₂)/(x₁-x₂)`. Skipping this branch produces a "division by zero polynomial" panic on the tiny `F_43` curve that has lots of these collisions; on a 256-bit curve you would basically never hit it, but the algorithm should still be correct.

Tests pin the basics: `P + (-P) = O`, doubling matches scalar-mul-by-2, scalars associate (`(a+b)P = aP + bP`), and `[ord]P = O` for the orders listed in the book.

### Chapter 15: Miller, Weil, Tate

Miller's algorithm is short — it is essentially square-and-multiply with a "line function" `h(P, Q)` accumulated alongside the point doubling/addition:

```rust
pub fn miller(
    ec: &PolyCurve, p: &PolyPoint, r: &PolyPoint, m: &Int,
) -> Polynomial {
    let mut t = p.clone();
    let mut f = one_poly();
    let mut bit = (m.significant_bits() - 2) as i64;
    while bit >= 0 {
        let h = hpq(ec, &t, &t, r);
        f = f.mul(&f, &ec.irrd, &ec.p);
        f = f.mul(&h, &ec.irrd, &ec.p);
        t = ec.add(&t, &t);
        if m.get_bit(bit as u32) {
            let h = hpq(ec, &t, p, r);
            f = f.mul(&h, &ec.irrd, &ec.p);
            t = ec.add(&t, p);
        }
        bit -= 1;
    }
    f
}
```

Weil is then four Miller invocations:

```rust
pub fn weil(
    ec: &PolyCurve, p: &PolyPoint, q: &PolyPoint, s: &PolyPoint, m: &Int,
) -> Polynomial {
    let qps = ec.add(q, s);
    let neg_s = PolyPoint::new(s.x.clone(), poly_neg(&s.y, &ec.p));
    let p_minus_s = ec.add(p, &neg_s);
    let t1 = miller(ec, p, &qps, m);
    let t2 = miller(ec, p, s, m);
    let t3 = miller(ec, q, &p_minus_s, m);
    let t4 = miller(ec, q, &neg_s, m);
    let w1 = t1.div(&t2, &ec.irrd, &ec.p);
    let w2 = t3.div(&t4, &ec.irrd, &ec.p);
    w1.div(&w2, &ec.irrd, &ec.p)
}
```

The Tate pairing is cheaper: two Miller calls plus one final exponentiation to `(p^k - 1)/m`. The tests check what every pairing should: bilinearity (`e(P, T+Q) = e(P,T)·e(P,Q)`) and that the result is an `m`-th root of unity.

### Chapters 16 + 17: print every point

These chapters build a 6-bit curve so deliberately small (`p=43`, `k=2`, total 1815 affine points) that you can enumerate every point on the extension curve and see the pairing land on each torsion subgroup explicitly.

The Weil demo enumerates every `(x, y)` on `E(F_{43^2})`, classifies each point as G1 (degree-0 coords) or G2 (degree-1 coords), and runs the bilinearity check across each subgroup product:

```
G1 x G1 test (order-11 in base subgroup) -----------
P: x = 3, y = 3
Q: x = 3, y = 40
T: x = 11, y = 11
S: x = 1, y = 18
  e(P, Q)    = 1
  e(P, T)    = 1
  e(P,T)*e(P,Q) = 1
  BILINEARITY HOLDS

G1 x G2 test (mixed) --------------------------------
P: x = 3, y = 3
Q: x = 0, y = 4*x^1 + 2
T: x = 0, y = 39*x^1 + 41
S: x = 1, y = 18
  e(P, Q)    = 34*x^1 + 19
  e(P, T)    = 9*x^1 + 28
  e(P, T+Q)  = 1
  e(P,T)*e(P,Q) = 1
  BILINEARITY HOLDS
```

This is the moment where the abstract definition of "bilinear pairing" clicks: within a single subgroup the pairing is trivial; *across* subgroups it produces a genuine element of `F_{43^2}*` that respects the group law on both arguments. Books explain this in two pages; watching the polynomial coefficients pop out is faster.

The Tate demo is the same idea with four arrangements (G1×G1, G1×G2, G2×G1, G2×G2).

### Chapter 18: BLS signatures

BLS is the cleanest payoff in the book — three primitives compose into something genuinely useful:

```rust
impl BlsSystem {
    pub fn keygen(&self) -> (Int, PolyPoint) {
        let sk = Modulus::new(&self.tor).rand();
        let pk = self.ex.mul(&self.g2, &sk);   // PK = sk · G2
        (sk, pk)
    }

    pub fn sign(&self, sk: &Int, msg: &[u8]) -> Option<Point> {
        let h = self.hash_to_g1(msg)?;
        Some(self.e.mul(&h, sk))               // σ = sk · H(m)
    }

    pub fn verify(&self, pk: &PolyPoint, msg: &[u8], sig: &Point) -> bool {
        let h = match self.hash_to_g1(msg) { Some(h) => h, None => return false };
        let w_sig = weil(&self.ex, &to_g2(sig), &self.g2, &self.aux, &self.tor);
        let w_pk  = weil(&self.ex, &to_g2(&h),  pk,       &self.aux, &self.tor);
        w_sig == w_pk                          // e(σ, G2) == e(H(m), PK)
    }
}
```

`hash_to_g1` is the part that hides the most subtlety: hash bytes to an `F_p` element, sweep `x` values until `f(x)` is a quadratic residue, then multiply the resulting point by the base-curve cofactor so it lands in the order-`r` subgroup. The cofactor multiplication is what separates a "point on the curve" from a "point in G1" — a detail that is easy to overlook until you wire it up.

Aggregation falls out of bilinearity for free:

```rust
let agg_sig = sys.aggregate_sigs(&[s1, s2, s3]);  // σ_agg = Σ σᵢ
let agg_pk  = sys.aggregate_pks(&[pk1, pk2, pk3]); // PK_agg = Σ PKᵢ
assert!(sys.verify(&agg_pk, msg, &agg_sig));       // one pairing check, n signers
```

This is the headline property of BLS that drives its use in Ethereum's consensus layer: 1000 signatures on the same message verify in *two* pairings instead of 2000. The demo runs three signers on the tiny curve and confirms the aggregate verifies.

The test suite covers seven cases: torsion-order invariants for G1/G2 and aggregated PKs, hash-to-G1 lands in the right subgroup, sign+verify roundtrip, rejection of tampered messages, rejection of wrong PKs, and the aggregate equation above.

### Chapter 6: more base curves

The first post only exposed one curve from the book — the 256-bit one used by ECDSA and DHKE. Chapter 6 actually ships four: 160-, 256-, 384- and 512-bit primes, each with its own generator. They're now all available:

```rust
pub fn curve_160() -> Curve { /* ac0…001, a4=1, a6=0x782e */ }
pub fn curve_256() -> Curve { /* 2b0…001, a4=1, a6=0xa87  */ }
pub fn curve_384() -> Curve { /* 2e0…001, a4=1, a6=0x310  */ }
pub fn curve_512() -> Curve { /* e20…001, a4=1, a6=0x41   */ }
```

Plus a parser for the labeled text format the book uses for these:

```rust
pub fn from_dat<P: AsRef<Path>>(path: P) -> io::Result<Self>
```

One test loads every `Curve_*_params.dat` shipped in `aux/` and verifies the constructed `Curve` matches the hard-coded constants exactly, so if the upstream changes its parameters we notice. Another test feeds an off-curve base point and confirms the loader rejects it instead of silently accepting a bad curve.

### Chapter 14: where pairing-friendly curves come from

This is the chapter that explains *how* something like the `curve_11` parameter file gets generated in the first place. Two pieces:

**Curve search** (`pairing_sweep_alpha.c` in the book, [`src/curve_search.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/curve_search.rs) here). For each embedding degree `k ∈ {5, 7, 11, 13, 17, 19, 23, 29, 31}`, sweep over (α, x) pairs and check whether the cyclotomic-style polynomial `Φ_{4k}(αx²)` produces a prime `r`. If so, evaluate the matching `q(z)` and `t(z)` from Freeman/Scott/Teske's taxonomy paper (algorithms 6.2 or 6.20, depending on `k`) and accept the candidate when `q` is also prime.

```rust
pub fn sweep(k: u32, lg2_r_max: u32) -> Vec<Candidate> { /* ... */ }

pub struct Candidate {
    pub k: u32,
    pub alpha: Int,
    pub x: Int,
    pub r: Int,   // torsion order (prime)
    pub q: Int,   // field prime
    pub t: Int,   // trace of Frobenius
    pub rho: f64, // log2(q)/log2(r), smaller is better
}
```

The output is sorted by ρ ascending — the closer to 1.0, the more efficient the curve. Tests verify the relation `(q + 1 − t) ≡ 0 (mod r)` for every candidate found (so the curve actually has a subgroup of order `r`).

**CM construction from a discriminant** (`get_curve.c`, [`src/cm_curve.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/cm_curve.rs)). Given a candidate from the sweep plus a discriminant `D = −|D|`:

1. Look up the *Hilbert class polynomial* `H_D(x)` from the book's precomputed table (`Hilbert_Polynomials.list`, ~150 entries).
2. Find a root `j` of `H_D` mod `p`. That root is the j-invariant of the curve we want.
3. Set `c = j / (1728 − j) mod p`, then `a4 = 3c`, `a6 = 2c`.
4. Validate: a random point `R` on `y² = x³ + a4·x + a6` should satisfy `[#E]·R = O`, where `#E = p + 1 − t`. If not, try the next j-root, or the curve's quadratic twist.

```rust
pub fn get_curve(
    hilbert: &HilbertTable,
    d_abs: u64,
    p: &Int,
    t: &Int,
) -> Option<Curve>
```

The Hilbert polynomials parser handles the book's standard text form (`x^n + c_{n-1}*x^{n-1} ... + c_0`); root finding handles arbitrary degree via Cantor–Zassenhaus over F_p. The recipe is the standard one: first isolate the linear factors `A(x) = gcd(x^p − x, H_D(x))`, then recursively split `A` by computing `gcd((x+r)^((p-1)/2) − 1, A)` for random `r` until each piece is degree 1 or 2 and the closed-form takes over.

The worked-example test pins it down: with `D = −11`, `p = 23`, `t = 9`, the algorithm produces `y² = x³ + 12x + 8 mod 23`, which has exactly `p + 1 − t = 15` points. By hand:

- `j(D=−11) = −32768` (from `H_D = x + 32768`)
- `j mod 23 = 7`, `1728 mod 23 = 3`, `c = 7 / (3 − 7) = 7 / −4 = 4 mod 23`
- `a4 = 3·4 = 12`, `a6 = 2·4 = 8`

The test then enumerates a random point and confirms `[15]·R = O`. End-to-end CM construction in ~150 lines of Rust.

### BLS on a real curve

The Chapter 18 demo above runs BLS on the 6-bit toy curve. With Chapter 14 in place, there's no reason to stop there — the same `curve_11_parameters.bin` the snark binary reads is now loadable directly into a `BlsSystem` without going through the (deliberately isolated) `snark/` module:

```rust
pub fn load_curve_params<P: AsRef<Path>>(path: P) -> io::Result<BlsSystem>
```

That binary file was originally generated by the chapter 14 algorithms — `pairing_sweep_alpha` found `(k=11, α, x, r, q, t)`, `get_curve` derived `(a4, a6)`, and the rest is system bookkeeping (irreducible, G2, cofactor). With the loader in place, a real-size BLS sign+verify runs in ~3.7s in release mode:

```rust
#[test]
#[ignore = "slow — uses real curve_11 system (~13s on a laptop)"]
fn test_bls_sign_verify_on_curve_11() {
    let sys = load_curve_params(CURVE_11_PATH).expect("load");
    let (sk, pk) = sys.keygen();
    let sig = sys.sign(&sk, b"production-sized message").expect("sign");
    assert!(sys.verify(&pk, b"production-sized message", &sig));
}
```

The "auxiliary" point the Weil pairing needs (a point of order coprime to the torsion `r`) is found by the cofactor-clearing trick: any point times `r` lands in the cofactor subgroup, which is automatically of order coprime to `r` for a pairing-friendly setup.

### Closing the loop: reproducing curve_11

If `curve_11_parameters.bin` was originally generated by the chapter 14 algorithms, can we reproduce it from scratch in the Rust port? The full pipeline — sweep → discriminant → HCP root → curve coefficients — has to actually invert the original generation.

The missing piece is recovering the discriminant from the publicly-known `(p, t)`. The CM condition gives a clean handle: `4p − t² = |D|·s²` for some integer `s ≥ 1`. Scan the Hilbert table, divide, and check whether the quotient is a perfect square:

```rust
pub fn find_cm_discriminant(
    hilbert: &HilbertTable,
    p: &Int,
    t: &Int,
) -> Option<u64>
```

Running this on curve_11 picks out **`D = -163`** — Heegner's largest class-number-1 discriminant, the one that famously makes `e^(π·√163)` look like an integer. Class number 1 means the Hilbert class polynomial is linear, with a single integer j-invariant: `H_{-163}(x) = x + 262537412640768000`. From there, `get_curve` reproduces the curve in ~2.3s in release mode, matching the order field from the binary:

```rust
#[test]
#[ignore = "slow"]
fn test_get_curve_reproduces_curve_11() {
    let sys = load_curve_params(CURVE_11_PATH).unwrap();
    let p = sys.e.modulus.clone();
    let t = (p.clone() + 1) - sys.e.order.clone();
    let h = hilbert();
    let d = find_cm_discriminant(&h, &p, &t).unwrap();   // 163
    let curve = get_curve(&h, d, &p, &t).unwrap();
    assert_eq!(curve.order, sys.e.order);
}
```

To make the whole pipeline runnable from the command line, [`bin/search`](https://github.com/sergey-melnychuk/ecc/blob/main/src/bin/search.rs) ties the pieces together — sweep, pick the best ρ, find the CM discriminant, build the curve, print parameters in the same labeled-text format as the book's `Curve_*_params.dat`:

```
$ cargo run --release --bin search -- 7 50
# Searching for pairing-friendly curves: k = 7, log2(r) ≤ 50

Found 4 candidate(s). Top by rho:
  [0] alpha = 11, x = 1, rho = 1.2381, |r| = 21b, |q| = 26b
  [1] alpha = 15, x = 3, rho = 1.2791, |r| = 43b, |q| = 55b
  [2] alpha = 135, x = 1, rho = 1.2791, |r| = 43b, |q| = 55b
  [3] alpha = 87, x = 1, rho = 1.2821, |r| = 39b, |q| = 50b

# Building a curve from the first viable candidate...
  [0] CM discriminant: D = -11

# Curve_k7_alpha11_x1_D11.dat

prime
37c467d
order
37c0d4c
cofactor
36
curve(a4   a6)
2f487bf
ceedab
basepoint(x   y)
0
b257f9
```

That's a curve nobody had before this run, with embedding degree 7, ρ ≈ 1.24, fresh out of the search.

### Bug fixes along the way

Porting forces you to look at the original code carefully, and a few real bugs surfaced in the core modules:

- **`P + (-P)` in [`elliptic.rs`](https://github.com/sergey-melnychuk/ecc/blob/main/src/elliptic.rs)** — the original `Curve::add` mishandled the `y₁ + y₂ = 0` branch and computed a nonsensical denominator. Now it returns the point at infinity (also covering 2-torsion doubling).
- **`Point::inf()` as `(0, 0)`** — the original encoded the identity as the affine origin, which aliases on any curve with `b = 0`. Now there is an explicit `inf: bool` flag.
- **`Modulus::rand` reseed** — `rug::RandState::new()` uses a fixed default seed, so successive calls returned the same value, breaking the zero-knowledge property of the random blinders in the SNARK prover. Now reseeds from `/dev/urandom` each call.
- **`Polynomial::degree()` underflow** — empty polynomials (post-`trim()`) underflowed `coef.len() - 1` on `usize`. Now uses `saturating_sub`.
- **`Polynomial::normal` / `inv` un-reduced coefficients** — these called `Modulus::inv` on the raw leading coefficient, which would panic when the value was `0 mod p` but stored as a nonzero `Int`. Now reduces mod p first.
- **`Modulus::sqrt` Tonelli–Shanks** — the slow path (the case `p ≡ 1 mod 4`) had *two* bugs that compounded into an `unreachable!()` panic: the non-residue search broke on residues instead of non-residues (inverted condition), and the inner loop's iteration variable was off-by-one. Neither was caught earlier because every previously-used curve had `p ≡ 3 mod 4` and took the fast path. The slow path was first exercised by loading `curve_11` for BLS, which has `p ≡ 1 mod 4`. Rewritten following the standard Wikipedia formulation, with a regression test on `p = 13`.
- **`poly_sqrt` Tonelli–Shanks lift** — applied the same fix to the polynomial version (`poly_elliptic.rs`). The lifted T-S still has a genuine edge case on very small fields: when the algorithm's `M` reaches 1 and the working `t` becomes -1, completing the sqrt would require multiplying by sqrt(-1) ∈ GF(p^k), which the standard T-S formulation doesn't expose. For the `F_43²` demo curve this happens often enough to matter, so a brute-force fallback (`O(p^k)`, only viable for pk ≤ 2²⁰) catches what T-S can't reduce. On production-sized curves T-S handles everything.
- **`gpow_p2` misleading docstring** — the comment claimed "Euler criterion" but the function computes `a^((p-1)/2)` with the *base* prime, not `(p^k-1)/2`. It's correct for its actual use case (the Cantor–Zassenhaus split step in F_p, which is now the only caller besides tests), but the docstring was lying. Rewritten to say what the function really does.

Every one of these has a test pinning the new behavior.

### Test coverage

```
$ cargo test --lib
test result: ok. 82 passed; 0 failed; 2 ignored
```

The two ignored tests are the real-curve BLS roundtrip and the curve_11 reproduction from `(D, p, t)` — each takes a few seconds in release mode and is gated behind `--ignored`. Run them explicitly with:

```
cargo test --lib --release -- --ignored
```

Each new module ships with its own tests:

| Module | Tests | Highlights |
|--------|-------|------------|
| `poly_elliptic` | 8 | doubling, identity, scalar associativity, order via factor sweep |
| `poly_elliptic::sqrt_test` | 2 | polynomial Tonelli–Shanks against random squares |
| `pairing` | 8 | cardinality formula, Weil/Tate bilinearity, alternating, torsion-root |
| `bls` | 9 | tiny + real-curve sign/verify, tamper rejection, key swap, aggregate |
| `curve_search` | 5 | Φ_{4k} sanity, the `(q+1-t) mod r = 0` relation, sweep finds candidates |
| `cm_curve` | 11 | HCP parse + lookup, CZ on cubic & quartic, `D=-11/p=23` curve, curve_11 |
| `elliptic` (new) | 6 | `[order]·G = O` for all four book curves, `.dat` loader round-trip |

Plus four new binaries:

```
cargo run --bin weil               # Weil pairing on every G1×G1, G1×G2, G2×G2 pair (tiny curve)
cargo run --bin tate               # Tate pairing, four configurations (tiny curve)
cargo run --bin bls                # BLS keygen/sign/verify + 3-signer aggregate (tiny curve)
cargo run --release --bin search   # full pipeline: sweep → CM disc → get_curve → .dat-style output
```

### The book is closed

That's all 19 chapters of *Elliptic Curve Cryptography for Developers* covered:

- **2** modular arithmetic; **3** elliptic curves; **4–5** DHKE + ECDSA
- **6** base curves at multiple security levels
- **7–12** polynomial arithmetic (mul-prep, Euclidean division, GCD, irreducibles, exponentiation)
- **13** elliptic curves over extension fields
- **14** pairing-friendly curve search + CM construction
- **15** Miller's algorithm; **16–17** Weil and Tate pairings
- **18** BLS signatures (tiny field + real `curve_11`)
- **19** Groth16-style SNARK

The repo as it stands is enough to follow the book end-to-end *and* build small but real protocols on top of it: ECDSA on a 256-bit curve, BLS on the book's `curve_11` parameters, a Groth16 SNARK on the book's circuit example, plus end-to-end pairing-friendly curve generation via `bin/search`. Every chapter has tests. Most chapters have a runnable binary.

Full code: [github.com/sergey-melnychuk/ecc](https://github.com/sergey-melnychuk/ecc).
