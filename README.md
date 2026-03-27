# Homomorphic Encryption over One-Time Pads: Research Summary

## Background

The researcher started with a Jupyter notebook implementing a "Fully Homomorphic Caesar Cipher" (https://github.com/MiscellaneousStuff/homomorphic-caesar-cipher/blob/main/main.ipynb). The core idea: use modular addition as encryption (like a Caesar cipher, but with OTP-sized random keys for information-theoretic security) and explore what computations can be performed on the encrypted data without decryption.

## What Works

### Addition (Proven, Composable, Unlimited Depth)

Encryption: `E(x) = (x + k) mod m`

Adding two encrypted values: `E(a) + E(b) = (a + b + ka + kb) mod m`

Alice tracks key accumulation through the compute graph and subtracts `ka + kb`. This composes to arbitrary depth. Alice just sums all the keys she used.

### Subtraction and Negation (Proven, Composable)

Negation: `(m - E(a)) mod m` gives `(-a - k) mod m`, Alice flips her correction key to `-k mod m`.

Subtraction = addition + negation. Also free and unlimited depth.

### Single-Depth Multiplication via One-Hot Lookup Tables (Proven)

This is the key innovation. The notebook implementation (Cell 8) works as follows:

**Setup:**
- Domain is small (e.g., 0-3)
- Multiplication table T is public: `T[i][j] = i * j`
- Values are encrypted as one-hot vectors with per-position OTP keys
- `one_hot(2, 4) = [0, 0, 1, 0]`
- `E_a = [K_a[0], K_a[1], 1+K_a[2], K_a[3]]` (position 2 has 1+key, rest are just keys)

**Server computation:**
```
Step 1: row[j] = sum(T[i][j] * E_a[i] for i in domain)   # select row using encrypted one-hot a
Step 2: result = sum(row[j] * E_b[j] for j in domain)     # select column using encrypted one-hot b
```

**Alice's correction:**
```
correction1 = sum(row[j] * K_b[j] for j in domain)        # remove K_b contribution
noise = [sum(T[i][j] * K_a[i] for i in domain) for j]     # compute K_a noise per column
correction2 = sum(noise[j] * b_oh[j] for j in domain)     # remove K_a leakage through b
decrypted = (result - correction1 - correction2) % mod
```

This works correctly for all input pairs. Verified in the notebook.

**Critical property:** The one-hot dot product is LINEAR in each encrypted input separately, but BILINEAR overall (two encrypted things multiply together). The linearity per-input means the key contamination is separable — Alice can compute each correction term because she knows the keys AND the plaintext inputs.

### Security Properties

- **Information-theoretic security** (Shannon perfect secrecy) — each one-hot position is encrypted under an independent random key
- **Immune to quantum computers** — no computational assumption, unconditional
- **Zero noise** — exact arithmetic, no failure probability
- **Trivial key sizes** — for domain m with n inputs: n × m random values (e.g., 10 inputs over 8-bit domain = ~20 KB, vs 16-320 MB for TFHE bootstrapping keys)

## The Composability Problem (The Wall)

### The Core Issue

When you expand `E_a[i] * E_b[j]` algebraically:

```
(a_oh[i] + K_a[i]) * (b_oh[j] + K_b[j])
= a_oh[i]*b_oh[j]           # Term 1: clean signal
+ a_oh[i]*K_b[j]            # Term 2: depends on plaintext a
+ K_a[i]*b_oh[j]            # Term 3: depends on plaintext b  
+ K_a[i]*K_b[j]             # Term 4: keys only, always removable
```

For gate 1: Alice knows a and b (she encrypted them), so she can compute terms 2, 3, and 4. ✓

For gate 2 (chained multiplication, e.g., `(a*b) * c`): The input is the intermediate result `a*b`. Alice would need to know `a*b` to compute the cross terms — but computing `a*b` is the whole point of the scheme. **Knowing the key of a multiplication output requires knowing the plaintext inputs, because the effective key is entangled with the message.**

Formally: after subtracting `ka*kb` (term 4), the residual is `ab + ka*b + kb*a`. The effective encryption key of the product is `ka*b + kb*a`, which depends on the plaintext values a and b. In a normal OTP, key and message are independent. Multiplication destroys that independence.

### Why This Is Fundamental

The cross term `kb*a` is literally a one-time pad encryption of `a` with key `kb`. Removing it IS decryption. Asking "how do I remove this cross term without knowing a" is equivalent to "how do I decrypt without decrypting."

This is the exact reason the FHE community needed lattice-based cryptography. LWE encryption (`a·s + e`) has algebraic structure in the secret `s` that provides handles for manipulation. Flat random OTP keys have no such structure.

## Ideas Explored and Their Status

### One-Hot Output (Proposed by researcher — WORKS for single depth)

Instead of outputting a scalar product, output a one-hot vector of the product. This gives the output the right SHAPE to feed into the next gate. The one-hot lookup mechanism doesn't require a clean one-hot input — E_a is already a noisy vector and the dot product still works.

**Status:** The shape composability works, but the algebraic cross-term problem remains. The output vector at each position v contains:
```
output[v] = [a*b == v ? 1 : 0] + f(a, K_b) + g(b, K_a) + h(K_a, K_b)
```
Terms f and g still depend on plaintext inputs.

### CRT Decomposition (Proposed by researcher — WORKS for representation)

Instead of representing a value x in {0,...,m-1} as a single one-hot of length m, decompose it via Chinese Remainder Theorem into multiple smaller one-hot vectors over coprime moduli.

Example: m=210 = 2×3×5×7
- x=17 → (17 mod 2, 17 mod 3, 17 mod 5, 17 mod 7) = (1, 2, 2, 3)
- Each residue encoded as a tiny one-hot vector
- Total representation: 2+3+5+7 = 17 values instead of 210
- Table cost: 2³+3³+5³+7³ = 503 entries instead of 9.2M

For 8-bit integers (domain 256), optimal factorisation {5,7,8} gives product=280, table cost=980 entries.

**Key insight from researcher:** This is analogous to sinusoidal position embeddings in transformers — multiple clocks at different frequencies exponentially increase representational capacity, exploiting the isomorphism between Z/NZ and the product of Z/p_iZ.

**Status:** CRT works perfectly for representation and parallel computation per residue channel. The cross-term problem still applies within each channel, but each channel is tiny.

### Table Rotation / Blind Rotation (Discussed — NOT portable to OTP)

Instead of one-hot selection, rotate the lookup table by the encrypted amount and read position 0. This is how TFHE works (polynomial ring rotation).

**Status:** Doesn't work with OTPs. The server can rotate by `E(a) = a + k`, but the answer ends up at position k, and extracting it requires sending the whole rotated table back to Alice. No compression advantage. The rotation trick requires the algebraic structure of polynomial rings that OTPs lack.

### Encrypted Sign Test / Comparison (Identified as the critical missing primitive)

If you had addition + subtraction + comparison, you could build conditional selection (mux). With mux + addition you can simulate any lookup table via linear scan. This would give multiplication without bilinear cross terms.

**Status:** Sign/comparison is inherently nonlinear. Addition, subtraction, and negation are all linear maps on Z/mZ. No composition of linear operations produces a nonlinear one. The only nonlinear operation in the scheme is the one-hot lookup, which is single-depth.

### Modulus Switching / Scaling (Proposed by researcher)

Encrypt as `d(k + m)` where d is a public denominator. After operations accumulate noise, divide by d to shrink everything.

**Status:** This is essentially the modulus switching technique from BFV/BGV, reinvented for the OTP setting. It manages noise magnitude but doesn't solve the cross-term problem. Noise management is problem B; cross-term correction requiring plaintext knowledge is problem A.

### Correction Table Approach (Proposed by researcher — PARTIALLY WORKS)

The most promising approach explored. After raw multiplication:
```
E(a)*E(b) = ab + ka*b + kb*a + ka*kb
```

1. Subtract `ka*kb` — always precomputable ✓
2. For `kb*a`: precompute correction table `F[i] = kb*i` for all i in domain. Select correct entry using E_a (original encrypted one-hot). This is a LINEAR operation against E_a (plaintext table × encrypted one-hot = no bilinear cross terms). Residual noise `sum_i(kb*i * K_a[i])` is fully known to Alice. ✓
3. Same for `ka*b` using E_b ✓

**The problem:** If the correction tables are plaintext, they leak the keys (F[1] = kb reveals kb). If encrypted under fresh OTP keys, selecting from them is bilinear again and produces new cross terms that need correction.

**Researcher's key insight:** The cross terms `ka*b` and `kb*a` are UNIVARIATE — each depends on only one unknown. The correction for each can be expressed as a lookup against the ORIGINAL encrypted one-hots E_a and E_b. Alice knows a and b for gate 1, so she can resolve all residuals.

### Recursive Correction Problem (Current state of research)

The correction approach works for a single gate. For chained gates:

1. Each encrypted correction table lookup is itself a multiplication (encrypted table × encrypted index)
2. Each produces its own cross terms
3. Correcting those requires more lookups
4. Which produce more cross terms

**However:** At each recursion level, the corrections are lookups against the ORIGINAL encrypted one-hots (E_a, E_b, E_c), which are clean inputs whose plaintexts Alice knows. So each correction level is gate-1-style.

**Open question:** Does this recursion converge? Is the total correction a finite, precomputable series? Or does each layer introduce irreducible new terms forever? The number of correction terms at each layer might be bounded and enumerable since Alice knows the full circuit, all keys, and all tables. This is the active research frontier.

## Comparison with TFHE-rs (Production FHE)

| Metric | TFHE-rs | OTP-OneHot Scheme |
|---|---|---|
| Security model | Computational (LWE) | Information-theoretic |
| Quantum resistant | Assumed | Proven |
| Key size (8-bit) | 16-320 MB | ~20 KB |
| Per-gate cost | ~10ms CPU, ~0.8ms GPU | ~μs (dot products) |
| Noise management | Required (bootstrapping) | None (exact arithmetic) |
| Failure probability | 2^-128 | 0 (exact) |
| Max native precision | ~7 bits | Any (table size) |
| Composable addition | Yes | Yes (unlimited) |
| Composable multiplication | Yes (via bootstrapping) | Single depth proven, chained is open problem |

## Key Insights from the Conversation

1. **Addition over OTP is trivially homomorphic** — well-known but the researcher independently derived it
2. **One-hot encoding linearises table lookups** — the dot product against an encrypted one-hot is linear per-input, making contamination separable
3. **CRT decomposition compresses one-hot representations exponentially** — from O(m) to O(sum of factors), analogous to sinusoidal position embeddings
4. **The multiplication cross-term problem is equivalent to "decrypting without decrypting"** — `kb*a` is literally an OTP encryption of a, and removing it requires knowing a
5. **The effective key of a multiplication output depends on the plaintext inputs** — this violates the key-message independence that defines OTP security, and is the fundamental reason OTP + multiplication is hard
6. **Correction tables can be selected using original encrypted one-hots** — this avoids bilinear cross terms in the correction step, but the correction tables themselves need encryption, leading to recursion
7. **TFHE exists because lattice structure provides the algebraic handles that flat random keys lack** — the researcher independently traced the exact path that led to TFHE's design

## Repository

https://github.com/MiscellaneousStuff/homomorphic-caesar-cipher

## Files

- `main.ipynb` — Original notebook with addition and single-depth multiplication implementations
- `fhe_explorer.jsx` — Interactive browser visualization of the one-hot multiplication scheme

## Active Research Direction

The recursive correction problem: can the infinite regression of correction lookups be collapsed into a finite, precomputable series that Alice evaluates once? Each recursion level uses lookups against original encrypted one-hots with known indices, so residuals at each level are deterministic functions of known quantities. The question is convergence — whether the total correction is a closed-form expression Alice can compute from her keys and the circuit structure alone.