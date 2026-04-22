# Ledger State

This note explains the ledger  Midnight's public state layer  and how it relates to private state.

> **Docs:** [Ledger ADT](https://docs.midnight.network/compact/reference/ledger-adt) · [Ledger State](https://docs.midnight.network/compact/reference/compact-reference#declaring-and-maintaining-public-state)
> **Examples:** [04.01 Commitment Pattern](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/04.01.commitment.compact)

---

## Intuition First

The ledger is Midnight's **public world**. Every node on the network stores it. Everyone can read it.

Private data (witnesses) lives on the user's local machine and never touches the chain. The two are connected by `disclose()`, a compile-time annotation that marks intentional disclosure.

The key insight: **privacy is the default, not opt-in.** Private data stays private unless you explicitly mark it for disclosure.

---

## The Two Worlds

| Property | `export ledger` | Private state (witnesses) |
|----------|----------------|--------------------|
| Where it lives | Every network node | User's local storage |
| Who can read it | Everyone | Only the owner |
| On-chain representation | Plaintext value | Nothing (commitment or nothing) |
| How it's updated | Via ZK proof | Never touches the chain |
| Update mechanism | Ledger assignment | Witness callbacks |

---

## Ledger State Updates

![Ledger Update Flow](../images/ledger_update_flow.png)

The ledger update happens atomically with the proof. Either the proof is valid and the state changes, or it isn't and nothing changes.

---

## Declaring Ledger Fields

```compact
ledger val: Field;                                    // basic field
export ledger cnt: Counter;                           // exported, readable
sealed ledger config: Uint<32>;                       // write-once
export sealed ledger mapping: Map<Boolean, Field>;  // exported + sealed
```

| Modifier | Meaning |
|----------|--------|
| `export` | Readable from TypeScript (your DApp) |
| `sealed` | Writeable only during initialization |

- `ledger` without modifiers = basic, non-exported field
- `export ledger` = readable by your DApp
- `sealed ledger` = writeable during initialization only
- `export sealed ledger` = both

All ledger fields initialize to their type's default (zero, empty, first variant, etc.). The constructor can override them.

---

## The `disclose()` Boundary

`disclose()` is a **compile-time annotation**, not encryption. It tells the compiler: "I am intentionally disclosing witness data."

```compact
// Without disclose(): compiler error
export circuit record(): [] {
  stored = getSecret();   // error: potential witness disclosure
}

// With disclose(): compiles
export circuit record(): [] {
  stored = disclose(getSecret());  // ok: declared
}
```

The compiler tracks witness data through every operation, arithmetic, type conversions, function calls. If witness data could reach the ledger without `disclose()`, you get a compiler error.

**What this means:** You cannot accidentally leak private data. The compiler enforces the privacy boundary.

---

## When to Use `export ledger`

**Good candidates for `export ledger`:**

| Candidate | Why it belongs on-chain |
|-----------|----------------------|
| Global invariants (total supply, reserve balance) | Everyone needs to see them |
| State flags others react to | Needed for coordination |
| Commitments to private values (the hash, not the value) | Proves existence without revealing |
| Data your frontend needs to read directly | Otherwise you can't display it |

**Bad candidates:**

| Candidate | Why it doesn't belong on-chain |
|-----------|------------------------|
| Per-user balances | Only one user cares |
| Personal data | Privacy violation |
| Any value belonging to only one user | No one else needs it |

**Heuristic:** If removing this field would break another user's ability to interact with the contract, it belongs in `export ledger`.

---

## The Commitment Pattern

![Commitment Pattern](../images/ledger_commitment_pattern.png)

If `export ledger` puts values on-chain as plaintext, and private state keeps values off-chain entirely, how do you verify something about private state?

**Answer: commitments.** Store the hash on-chain. Keep the value off-chain. Prove knowledge of the value without revealing it.

```compact
export ledger balanceCommitments: Map<Bytes<32>, Bytes<32>>;

export circuit commitBalance(value: Uint<64>): [] {
  const nonce = freshNonce();
  const commitment = persistentCommit<Uint<64>>(value, nonce);
  balanceCommitments.insert(disclose(callerAddress()), disclose(commitment));
}
```

**Critical:** The nonce must never be reused. Two commitments with the same nonce and value are identical on-chain.

---

## Commitment Tools

| Function | Output | Persists? | For ledger? | Witness-tainted? |
|----------|--------|-----------|------------|------------------|-----------------|
| `transientHash` | `Field` | No | No | Yes |
| `transientCommit` | `Field` | No | No | **No** |
| `persistentHash` | `Bytes<32>` | Yes | Yes | Yes |
| `persistentCommit` | `Bytes<32>` | Yes | Yes | **No** |

- **Persistent:** Survives contract upgrades. Use for long-term storage.
- **Transient:** Does not survive upgrades. Use for temporary computations.
- **Commit (vs hash):** Includes a random nonce. The nonce hides the input even if the value is known. Use when the value might be guessable.

---

## Ledger-State Types

| Type | What it is |
|------|-----------|
| `T` (any type) | A single `Cell<T>`, readable and writable |
| `Counter` | Unsigned counter with atomic increment (low contention) |
| `Set<T>` | Unbounded set of unique values |
| `Map<K, V>` | Unbounded key-value mapping |
| `List<T>` | Unbounded ordered list (pushFront/popFront) |
| `MerkleTree<n, T>` | Bounded Merkle tree of depth n (2 ≤ n ≤ 32) |
| `HistoricMerkleTree<n, T>` | Like `MerkleTree` but retains past roots |
| `Kernel` | Built-in operations (block time, tokens, address) |

---

## Choosing the Right Type

| Use case | ADT |
|---------|-----|
| Single mutable value | `ledger f: T` (Cell) |
| Monotonically growing counter (low contention) | `Counter` |
| Membership tracking | `Set<T>` |
| Per-key storage | `Map<K, V>` |
| Ordered queue | `List<T>` |
| ZK membership proofs (current root only) | `MerkleTree<n, T>` |
| ZK membership proofs (any past root) | `HistoricMerkleTree<n, T>` |
| Block time, tokens, contract address | `Kernel` |

---

## Common Mistakes

1. **Treating `export ledger` as encrypted.** It isn't. Everything in `export ledger` is plaintext and readable by everyone. If you need privacy, use the commitment pattern.

2. **Forgetting that witnesses never touch the chain.** Witness data stays local. Only the proof goes on-chain. You cannot store witness data directly, you must `disclose()` it first.

3. **Reusing nonces.** Two commitments with the same nonce and value are identical on-chain. Always use a fresh nonce.

4. **Putting per-user data in the ledger.** If only one user cares about a value, it shouldn't be in the ledger. It's a privacy leak.

5. **Using `transientHash` for ledger storage.** Transient values don't survive contract upgrades. Use `persistentHash` or `persistentCommit`.

---

## Comparison Layer

| Concept | Solidity | Rust | Compact |
|---------|---------|------|--------|
| Public state | `uint256 publicVar` | storage fields | `export ledger f: T` |
| Private state | `private uint256` (still on chain) | `u64` in memory | `witness` (stays local) |
| State updates | direct assignment | `storage.write()` | via ZK proof |
| Reading state | `Contract.state()` | direct read | `ledger(state)` |

---

## Quick Recap

- The ledger is public on-chain state. Everyone can read it.
- Private data (witnesses) stays local. Only the proof goes on-chain.
- `disclose()` marks intentional disclosure. It's a compile-time annotation, not encryption.
- Store commitments on-chain. Keep values off-chain. Prove knowledge without revealing.
- Always use a fresh nonce for commitments. Reuse = privacy leak.
- `transient*` doesn't survive upgrades. `persistent*` does.
- `transientCommit` can commit private values without `disclose()`. The random nonce provides sufficient hiding.

---

## Cross-Links

- **Previous:** [Writing a Contract](./chapter-03.md)  Contract structure
- **Next:** [Circuits](./chapter-05.md)  How circuits work
- **See also:** [Explicit Disclosure](./chapter-07.md)  The disclose() boundary in depth
- **See also:** [Ledger ADTs](./chapter-09.md)  Map, Set, MerkleTree details
- **Examples:** [04.01 Commitment Pattern](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/04.01.commitment.compact)