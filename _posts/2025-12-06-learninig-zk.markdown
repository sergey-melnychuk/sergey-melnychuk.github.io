---
layout: post
title: "Learning Zero-Knowledge Proofs"
date: 2025-12-06 12:34:56 +0100
categories: rust zkp zero-knowledge-proofs sudoku circom noir halo2
---

TL/DR: [ZKP](https://github.com/sergey-melnychuk/ZKP) - a learning journey through zero-knowledge proof systems: Groth16, PLONK, and Halo2. Same problem, three frameworks, one surprising insight.

### How it started

After working on [Beerus]({{ site.url }}{% post_url 2024-12-30-beerus-in-2024 %}) and spending a lot of time around Starknet and Ethereum infrastructure, I kept bumping into zero-knowledge proofs from the outside. I understood what they do at a high level - prove that you know something without revealing it - but I did not understand *how* they actually work. The math behind R1CS, QAP, polynomial commitments and pairings was a black box to me, and I wanted to open it.

I decided to learn by building: pick a simple problem, implement it end-to-end in one framework, then do it again in another, and then in a third one. Compare everything. The idea was that implementing the same circuit in multiple proof systems would reveal what is fundamental vs what is framework-specific.

### Phase 1: Sudoku and Groth16

The classic starting point. Sudoku is a perfect first ZK circuit because the rules are simple to express as constraints: each cell is 1-9, each row/column/block sums to 45, and the solution must match the given clues.

```
template InRange() {
    signal input in;
    signal output out;

    signal temp[8];

    temp[0] <== in - 1;
    temp[1] <== temp[0] * (in - 2);
    temp[2] <== temp[1] * (in - 3);
    // ... all the way to 9
    out <== temp[7] * (in - 9);

    out === 0;
}
```

This is Circom, and it reads like a DSL for arithmetic constraints. The `InRange` template is a nice trick: `(x-1)(x-2)...(x-9) = 0` is only satisfied when `x` is one of 1..9. No branching, no comparison operators - just polynomial arithmetic. That's the first lesson of ZK circuits: everything is polynomials.

The clue-matching constraint is even more elegant:

```
puzzle[i] * (solution[i] - puzzle[i]) === 0;
```

If `puzzle[i]` is 0 (empty cell), the constraint is trivially satisfied. If it is non-zero (a given clue), then `solution[i]` must equal `puzzle[i]`. One line, no `if/else`.

The whole Groth16 workflow is a pipeline:

```bash
# Compile circuit to R1CS + WASM witness generator
circom sudoku.circom --r1cs --wasm --sym

# Trusted setup ceremony (Powers of Tau)
snarkjs powersoftau new bn128 12 pot12_0000.ptau -v
snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau \
  --name="1st contribution" -v -e="random"
snarkjs powersoftau prepare phase2 pot12_0001.ptau pot12_final.ptau -v

# Circuit-specific setup
snarkjs groth16 setup sudoku.r1cs pot12_final.ptau sudoku_0000.zkey
snarkjs zkey contribute sudoku_0000.zkey sudoku_final.zkey \
  --name="1st Contributor" -v -e="random"

# Prove and verify
snarkjs groth16 prove sudoku_final.zkey witness.wtns proof.json public.json
snarkjs groth16 verify verification_key.json public.json proof.json
```

The trusted setup is the part that makes Groth16 both impressive and awkward. The proof is tiny (192 bytes!) and verification is fast (~5ms), but you need this ceremony for every circuit. If the ceremony is compromised, anyone can forge proofs. That is a serious trade-off.

### Verifying Groth16 from Rust

To make sure I actually understand what is happening under the hood, I implemented a Groth16 verifier in Rust using [arkworks](https://github.com/arkworks-rs). This was a natural next step from [learning ECDSA with arkworks]({{ site.url }}{% post_url 2023-11-22-rust-arkworks-ecdsa %}) earlier.

The core of Groth16 verification is one pairing equation:

```rust
fn verify_groth16(proof: &Proof, vk: &VerificationKey, public_inputs: &[Fr]) -> bool {
    let proof_a = parse_g1(&proof.pi_a);
    let proof_b = parse_g2(&proof.pi_b);
    let proof_c = parse_g1(&proof.pi_c);

    let vk_alpha = parse_g1(&vk.vk_alpha_1);
    let vk_beta = parse_g2(&vk.vk_beta_2);
    let vk_gamma = parse_g2(&vk.vk_gamma_2);
    let vk_delta = parse_g2(&vk.vk_delta_2);

    // Compute public input commitment
    let mut vk_x = parse_g1(&vk.ic[0]);
    for (i, input) in public_inputs.iter().enumerate() {
        let ic_point = parse_g1(&vk.ic[i + 1]);
        let scaled = ic_point.mul_bigint(input.into_bigint());
        vk_x = (vk_x + scaled).into();
    }

    // Verify: e(A, B) = e(α, β) · e(vk_x, γ) · e(C, δ)
    let result = Bn254::pairing(proof_a, proof_b)
        + Bn254::pairing(-vk_alpha, vk_beta)
        + Bn254::pairing(-vk_x, vk_gamma)
        + Bn254::pairing(-proof_c, vk_delta);

    result.is_zero()
}
```

This was the moment when the abstract math clicked. A pairing `e(P, Q)` maps two elliptic curve points to a target group element, and the bilinearity property `e(aP, bQ) = e(P, Q)^(ab)` is what makes the whole thing work. The prover encodes the computation in curve points, and the verifier checks one equation that could only be satisfied if the prover actually did the work. All of the R1CS, QAP, and polynomial evaluation machinery is there to make sure the prover can't cheat.

### Phase 2: Merkle tree membership - three frameworks

For the second problem I chose Merkle tree membership proofs: "I know a secret that hashes to a leaf in this tree." This is the building block behind mixers like Tornado Cash and pretty much every privacy-preserving protocol. And I implemented it three times.

#### Circom (Groth16)

```
template MerkleProof() {
    signal input secret;
    signal input siblings[4];
    signal input pathIndices[4];
    signal input root;
    signal input nullifier;

    component leafHasher = Poseidon(1);
    leafHasher.inputs[0] <== secret;

    component hashers[4];
    signal hashes[5];
    signal lefts[4];
    signal rights[4];

    hashes[0] <== leafHasher.out;

    for (var i = 0; i < 4; i++) {
        hashers[i] = Poseidon(2);
        lefts[i] <== hashes[i] - pathIndices[i] * (hashes[i] - siblings[i]);
        rights[i] <== siblings[i] + pathIndices[i] * (hashes[i] - siblings[i]);

        hashers[i].inputs[0] <== lefts[i];
        hashers[i].inputs[1] <== rights[i];
        hashes[i + 1] <== hashers[i].out;
    }

    root === hashes[4];

    component nullHasher = Poseidon(1);
    nullHasher.inputs[0] <== secret;
    nullifier === nullHasher.out;
}
```

The conditional swap (`lefts[i] <== hashes[i] - pathIndices[i] * (hashes[i] - siblings[i])`) is the same trick as in Sudoku: no branching, just arithmetic that evaluates to the right thing depending on whether `pathIndices[i]` is 0 or 1. This is Poseidon hashing, which is ZK-friendly (~200 constraints per hash vs ~25,000 for SHA-256).

Result: 2,387 constraints.

#### Noir (PLONK)

```rust
use dep::poseidon::poseidon2::Poseidon2;

fn main(
    secret: Field,
    siblings: [Field; 3],
    path_indices: [u1; 3],
    root: pub Field,
    nullifier: pub Field
) {
    let leaf = Poseidon2::hash([secret], 1);

    let computed_nullifier = Poseidon2::hash([secret], 1);
    assert(nullifier == computed_nullifier);

    let mut current_hash = leaf;
    for i in 0..3 {
        let is_left = path_indices[i];
        let (left, right) = if is_left == 0 {
            (current_hash, siblings[i])
        } else {
            (siblings[i], current_hash)
        };
        current_hash = Poseidon2::hash([left, right], 2);
    }
    assert(current_hash == root);
}
```

This is Noir - Aztec's high-level language for ZK circuits. It reads almost like Rust. The same logic, but instead of manually constructing constraint arithmetic for conditional swaps, you just write `if/else` and the compiler handles it. The circuit description went from 46 lines of Circom to 48 lines of almost-Rust.

Result: 24 opcodes. That is **99x fewer constraints** than the Groth16 version.

Wait. 99x. That number stopped me in my tracks.

#### The 99x surprise

Both circuits do the exact same thing: hash a secret, climb a Merkle tree, check the root. The difference is that PLONK uses custom gates that can express Poseidon hashing far more efficiently than Groth16's R1CS. In R1CS every constraint is of the form `a * b = c`, and expressing a Poseidon round in that format takes hundreds of constraints. PLONK's custom gates can handle the entire Poseidon round in a few constraints.

This was the biggest insight of the whole learning journey: the choice of proof system has a *massive* impact on circuit size, and circuit size directly affects proving time. It is not just a theoretical difference - it is nearly two orders of magnitude.

#### Halo2 (PLONK, low-level)

And then I went to the deep end: Halo2, Zcash's low-level PLONK implementation. Same Merkle proof circuit, but now I'm defining custom gates by hand.

```rust
fn configure_swap_gate(&self, meta: &mut ConstraintSystem<pallas::Base>) {
    meta.create_gate("swap", |meta| {
        let s = meta.query_selector(self.selector);
        let current = meta.query_advice(self.advices[5], Rotation::cur());
        let sibling = meta.query_advice(self.advices[6], Rotation::cur());
        let path_index = meta.query_advice(self.advices[7], Rotation::cur());
        let left = meta.query_advice(self.advices[8], Rotation::cur());
        let right = meta.query_advice(self.advices[9], Rotation::cur());

        // path_index must be 0 or 1
        let bool_check = path_index.clone()
            * (Expression::Constant(pallas::Base::ONE) - path_index.clone());

        // Conditional selection via arithmetic
        let expected_left = current.clone()
            * (Expression::Constant(pallas::Base::ONE) - path_index.clone())
            + sibling.clone() * path_index.clone();
        let expected_right = sibling
            * (Expression::Constant(pallas::Base::ONE) - path_index.clone())
            + current * path_index;

        vec![
            s.clone() * bool_check,
            s.clone() * (left - expected_left),
            s * (right - expected_right),
        ]
    });
}
```

This is where you truly see how ZK circuits work at the lowest level. Every gate returns constraint polynomials that must evaluate to zero. The `path_index * (1 - path_index) = 0` constraint forces a binary value. The conditional selection `left = current * (1 - path_index) + sibling * path_index` implements if/else purely through arithmetic. There is no branching anywhere in the computation - only polynomials.

The full Halo2 circuit is 360 lines of Rust. The same logic that is 48 lines in Noir. But writing it taught me more about how PLONK actually works than any paper or tutorial could.

### Poseidon: why ZK has its own hash functions

One thing that confused me early on: why not just use SHA-256? Everybody knows SHA-256, it is battle-tested, why invent new hash functions?

The answer is constraint cost. SHA-256 uses bitwise operations (AND, XOR, rotation) that are extremely cheap on CPUs but extremely expensive in arithmetic circuits. Expressing a single SHA-256 round in R1CS takes thousands of constraints. Poseidon, on the other hand, is designed from the ground up for arithmetic circuits: it operates directly on field elements using additions and exponentiations, which are native operations in the constraint system. The result: ~200 constraints for a Poseidon hash vs ~25,000 for SHA-256. That is a 125x difference, and in ZK circuits, fewer constraints means faster proving.

### The nullifier pattern

Both the Circom and Noir Merkle circuits include a nullifier: `nullifier = Hash(secret)`. The nullifier is a public output that can be tracked on-chain to prevent double-spending. You prove you know a secret in the tree (without revealing which leaf), and the nullifier ensures you can only do it once. If you try to prove again with the same secret, the same nullifier shows up and gets rejected.

This is the fundamental building block behind privacy-preserving protocols: a public identifier derived from a private secret, linkable across uses but not back to the original secret. Simple in concept, elegant in implementation.

### What I learned

The most valuable outcome was not any single circuit but the comparative understanding from implementing the same problem three times:

**Groth16** gives you the smallest proofs and fastest verification, at the cost of a per-circuit trusted setup. It is great when the circuit is fixed and verification cost matters (on-chain verification, for example). The R1CS constraint model is simple to understand but verbose to write.

**PLONK (via Noir)** gives you a universal setup (one ceremony for all circuits), dramatically fewer constraints thanks to custom gates, and a high-level language that reads like normal code. The 99x constraint reduction over Groth16 for the same computation was the single most surprising finding.

**Halo2** gives you maximum control over the circuit layout, custom gate design, and constraint system. It is the most verbose and hardest to use, but writing it forces you to understand exactly what is happening at every level. The Pasta curves (Pallas/Vesta) enable efficient recursion, which is why Zcash uses it.

Building the Groth16 verifier in Rust with arkworks was essential for understanding pairings. The math is not as scary as it looks from the outside - once you see `e(A, B) = e(α, β) · e(vk_x, γ) · e(C, δ)` in working code, the intuition follows.

The biggest lesson: ZK is not one thing. The choice between Groth16, PLONK, STARKs, and their variants is a real engineering decision with massive implications for proof size, proving time, verification cost, and setup trust assumptions. Understanding these trade-offs requires building with each one.

Full code and extensive documentation: [github.com/sergey-melnychuk/ZKP](https://github.com/sergey-melnychuk/ZKP)
