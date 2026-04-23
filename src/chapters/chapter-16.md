# Security and Best Practices

This note covers how to keep data private in Compact contracts, the tools, the patterns, and the mistakes that break privacy.

> **Docs:** [Keeping Data Private](https://docs.midnight.network/concepts/how-midnight-works/keeping-data-private) · [Basic Confidentiality](https://docs.midnight.network/compact/reference/writing#basic-confidentiality)
> **Examples:** [15.01 Hash Auth](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/15.01.hash-auth.compact) · [15.02 Merkle Auth](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/15.02.merkle-auth.compact) · [15.03 Nullifier](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/15.03.nullifier.compact)

---

## Intuition First

On Midnight, almost everything is potentially visible:

- Every argument to a ledger operation is public.
- Every read and write of a ledger field is public.
- Even function calls that look internal can leak data through their arguments.

The exceptions are narrow: `MerkleTree` insertions don't reveal the inserted value, and `transientCommit` with a fresh nonce doesn't carry witness taint. Everything else is visible.

This means privacy is not a default, it's a design discipline. You have to choose the right patterns deliberately.

---

## What's Publicly Visible

| Operation | What it reveals |
|-----------|----------------|
| `ledger.insert(v)` | The value `v` |
| `ledger.lookup(k)` | The key `k` and the returned value |
| `set.member(f(x))` | `f(x)`, not `x` |
| `merkleTree.insert(v)` | Does NOT reveal `v` |
| Circuit arguments | All of them |
| `witness` return values | Nothing (stays local) |

**The rule:** If it goes through the ledger or circuit arguments, assume it's public. The burden of proof is on privacy.

---

## Pattern 1: Hashes and Commitments

Store a hash or commitment instead of the value itself.

### When to Use Each

| Tool | When to use it |
|------|---------------|
| `persistentHash<T>(v)` | Identity and keys stored on-chain. Value space is large enough that brute-force is infeasible. |
| `persistentCommit<T>(v, rand)` | Sensitive values where the same value might appear multiple times (prevents correlation) or the value space is small (prevents guessing). |

### Why Commitment Over Hash

A bare hash of a small value space, like a vote for one of two candidates, can be brute-forced in seconds. The nonce in `persistentCommit` makes it infeasible even for small values.

```compact
// Bad, brute-forcible
commitment = persistentHash<Uint<1>>(vote);

// Good, nonce prevents brute force
nonce = freshNonce();
commitment = persistentCommit<Uint<1>>(vote, nonce);
```

### Nonce Reuse Is a Privacy Catastrophe

Two commitments with the same nonce and value are identical on-chain. If anyone knows the value, they can identify every transaction that used the same nonce.

**Rule:** Every commitment needs a fresh nonce. One safe approach: derive the nonce from a secret key and a round counter.

```compact
circuit deriveNonce(sk: Bytes<32>, round: Field): Field {
  return transientHash<Vector<2, Bytes<32>>>(
    [pad(32, "nonce:"), sk]
  );
}
```

---

## Pattern 2: Hash-Based Authentication

ZK proofs can emulate signatures using only hashes. Store a hash of the secret key as the "public key", circuits prove knowledge of the preimage without revealing it.

```compact
witness secretKey(): Bytes<32>;
export ledger organizer: Bytes<32>;
export ledger restrictedCounter: Counter;

constructor() {
  organizer = disclose(publicKey(secretKey()));
}

export circuit increment(): [] {
  assert(organizer == publicKey(secretKey()), "not authorized");
  restrictedCounter.increment(1);
}

pure circuit publicKey(sk: Bytes<32>): Bytes<32> {
  return persistentHash<Vector<2, Bytes<32>>>(
    [pad(32, "some-domain-separator"), sk]
  );
}
```

This pattern:
- Proves the caller knows the secret key.
- Doesn't reveal the secret key.
- Doesn't require a full signature scheme.

**Domain separator matters.** The same secret key can produce different public keys for different purposes. Never reuse a public key across different domains.

---

## Pattern 3: Merkle Trees for Anonymous Membership

A `MerkleTree` proves that a value exists in a set **without revealing which value**. This is the key difference from `Set`:

| Structure | What it proves | What it reveals |
|----------|---------------|----------------|
| `Set<Bytes<32>>` | Membership of a specific commitment | Which commitment was checked |
| `MerkleTree<n, T>` | Membership of a value | Only that *some* value was proven |

```compact
import CompactStandardLibrary;

export ledger items: MerkleTree<10, Field>;
witness findItem(item: Field): MerkleTreePath<10, Field>;

export circuit insert(item: Field): [] {
  items.insert(disclose(item));
}

export circuit check(item: Field): [] {
  const path = findItem(item);
  assert(
    items.checkRoot(merkleTreePathRoot<10, Field>(path)),
    "path must be valid"
  );
}
```

The TypeScript side provides the path:

```typescript
function findItem(context: WitnessContext, item: bigint): MerkleTreePath<bigint> {
  return context.ledger.items.findPathForLeaf(item)!;
}
```

**Depth choice:** Each level adds 1 to the circuit depth. Use the minimum depth that fits your use case. 16–20 is typical.

### When to Use HistoricMerkleTree

| Tree type | When to use |
|----------|------------|
| `MerkleTree` | Only current root matters. Proofs must verify against today's root. |
| `HistoricMerkleTree` | Proofs must verify against past roots. Used in nullifier patterns. |

Avoid `HistoricMerkleTree` when items are frequently removed, stale proofs could be accepted after the tree has changed.

### Path Performance

| Method | Complexity | Requires |
|--------|-----------|---------|
| `pathForLeaf` | O(1) | Knowing the leaf index |
| `findPathForLeaf` | O(n) scan | Scanning the tree |

Use `pathForLeaf` when you know the index. Use `findPathForLeaf` only when you don't.

---

## Pattern 4: Commitment/Nullifier

This pattern enables single-use anonymous authentication tokens, the core of Zcash and Zswap. It has four steps:

```
1. Insert a COMMITMENT (hash of secret data) into a MerkleTree
        ↓
2. To use the token: prove membership in the tree
        ↓
3. Add the NULLIFIER (different hash of the same secret) to a Set
        ↓
4. Assert the nullifier is NOT in the Set → prevents reuse
        ↓
The Set reveals SOME token was used, but NOT which one
```

The key insight: the `Set` of nullifiers is public and reveals only that *a* token was spent, not *which* one. The commitment's anonymity comes from the Merkle tree.

**Critical: commitment and nullifier must use different domain separators.** If they share a domain, they could be equal for some inputs, leaking the secret.

```compact
witness findAuthPath(pk: Bytes<32>): MerkleTreePath<10, Bytes<32>>;
witness secretKey(): Bytes<32>;

export ledger authorizedCommitments: HistoricMerkleTree<10, Bytes<32>>;
export ledger authorizedNullifiers: Set<Bytes<32>>;
export ledger restrictedCounter: Counter;

export circuit addAuthority(pk: Bytes<32>): [] {
  authorizedCommitments.insert(disclose(pk));
}

export circuit increment(): [] {
  const sk = secretKey();
  const authPath = findAuthPath(publicKey(sk));
  assert(
    authorizedCommitments.checkRoot(merkleTreePathRoot<10, Bytes<32>>(authPath)),
    "not authorized"
  );
  const nul = nullifier(sk);
  assert(!authorizedNullifiers.member(nul), "already incremented");
  authorizedNullifiers.insert(disclose(nul));
  restrictedCounter.increment(1);
}

pure circuit publicKey(sk: Bytes<32>): Bytes<32> {
  return persistentHash<Vector<2, Bytes<32>>>(
    [pad(32, "commitment-domain"), sk]
  );
}

pure circuit nullifier(sk: Bytes<32>): Bytes<32> {
  return persistentHash<Vector<2, Bytes<32>>>(
    [pad(32, "nullifier-domain"), sk]
  );
}
```

### Walkthrough

1. **Setup:** An authority inserts their public key into the tree.
2. **Authenticate:** The user proves membership via the Merkle path.
3. **Spend:** The nullifier is added to the `Set`. Next time, the `Set.member` check fails, reuse is prevented.
4. **Anonymity:** The `Set` of nullifiers is public. Someone spent a token. No one knows which one.

---

## Common Mistakes

1. **Using `persistentHash` for small value spaces.** A hash of a vote can be brute-forced. Always use `persistentCommit` with a fresh nonce for sensitive or small values.

2. **Reusing nonces.** Every commitment with the same nonce and value is identical on-chain. Derive nonces from a secret or counter.

3. **Using `Set` when you need anonymity.** `set.member(commitment)` reveals which commitment was checked. Use `MerkleTree` + `merkleTreePathRoot` when anonymity matters.

4. **Same domain for commitment and nullifier.** If they share a domain, they could collide for some inputs. Always use different domain separators.

5. **`transientHash` instead of `transientCommit`.** `transientHash` carries witness taint, it requires `disclose()` to store. `transientCommit` doesn't.

6. **Forgetting `disclose()` on Merkle tree inserts.** Even though `MerkleTree` doesn't reveal the value, the insert itself is a ledger operation. The value must be disclosed before insertion.

7. **Using `HistoricMerkleTree` unnecessarily.** It retains past roots, which costs storage and complexity. Use `MerkleTree` unless you need historical proofs.

---

## Privacy Tool Selection

| Goal | Tool |
|------|------|
| Hide a value on-chain | `persistentCommit` + fresh nonce |
| Prevent correlation of equal values | `persistentCommit` + fresh nonce |
| Authenticate without a full signature | `persistentHash` as public key |
| Prove set membership anonymously | `MerkleTree` + `merkleTreePathRoot` |
| Prove membership against past state | `HistoricMerkleTree` |
| Single-use anonymous token | Commitment/nullifier pattern |
| Temporary computation (no ledger) | `transientCommit` (no `disclose()` needed) |

---

## Quick Recap

- Almost everything on-chain is public. Assume ledger operations and circuit arguments are visible.
- Use `persistentCommit` over `persistentHash` for sensitive or small values.
- Never reuse a nonce. Derive it from a secret or counter.
- `MerkleTree` provides anonymity. `Set` does not.
- Commitment/nullifier requires different domain separators.
- `transientCommit` with a fresh nonce doesn't carry witness taint, no `disclose()` needed.
- Use `findPathForLeaf` (O(n)) only when you don't know the index. Use `pathForLeaf` (O(1)) when you do.

---

## Cross-Links

- **Previous:** [Testing and Debugging](./chapter-14.md)  Version management
- **See also:** [Ledger State](./chapter-04.md)  Commitment patterns
- **See also:** [Explicit Disclosure](./chapter-07.md)  Disclosure boundary
- **See also:** [Standard Library](./chapter-10.md)  Hash functions
- **See also:** [Example Projects](./chapter-16.md)  Working contracts
- **Examples:** [15.01 Hash Auth](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/15.01.hash-auth.compact) · [15.02 Merkle Auth](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/15.02.merkle-auth.compact) · [15.03 Nullifier](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/15.03.nullifier.compact)