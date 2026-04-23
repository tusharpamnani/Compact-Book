# Ledger ADTs

This note covers the collection types available for on-chain state, when to use each and the tradeoffs.

> **Docs:** [Ledger ADT](https://docs.midnight.network/compact/reference/ledger-adt)
> **Examples:** [09.01 ADTs](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/09.01.adts.compact) · [09.02 Merkle Trees](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/09.02.merkle.compact) · [09.03 Kernel](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/09.03.kernel.compact)

---

## Intuition First

Ledger ADTs are the data structures you use for on-chain state. They're not like in-memory data structures in TypeScript, they're persistent, on-chain, and their operations produce verifiable state transitions.

Each type optimizes for a specific access pattern:

- **Counter**, Low-contention atomic counters
- **Set**, Membership tracking
- **Map**, Key-value storage
- **List**, Ordered queue
- **MerkleTree**, Membership proofs with a single root
- **HistoricMerkleTree**, Membership proofs across time

The choice matters because the on-chain representation affects what your circuits can prove.

---

## Counter

A simple unsigned counter. The key advantage over `Cell<Uint<64>>` is that `Counter` does not read the current value when incrementing, so transactions are less likely to be rejected due to contention.

```compact
ledger votes: Counter;

export circuit vote(): [] {
  votes += 1;
}

export circuit retract(): [] {
  votes -= 1;
}

export circuit getVotes(): Uint<64> {
  return votes;
}
```

Operations:

| Operation | Syntax | What it does |
|----------|--------|-----------|
| Increment | `field += n` | Add `n`, no read required |
| Decrement | `field -= n` | Subtract `n`, errors if negative |
| Read | `field` | Returns as `Uint<64>` |
| Compare | `field.lessThan(n)` | Returns `Boolean` |

**Why low contention?** A normal `Uint<64>` read-modify-write has three steps: read the value, add one, write back. Under concurrent calls, two transactions might read the same value and both write the same new value. `Counter` combines these steps atomically.

---

## Cell (Plain Types)

When you declare `ledger f: T` without a collection type, it's implicitly a `Cell<T>`. Read and write operations use shorthand.

```compact
ledger owner: Bytes<32>;

export circuit setOwner(addr: Bytes<32>): [] {
  owner = disclose(addr);  // shorthand for owner.write(...)
}

export circuit getOwner(): Bytes<32> {
  return owner;  // shorthand for owner.read()
}
```

`Cell<T>` is the default. Use it for single, mutable values.

---

## Set

An unbounded set of unique values. Inserting a duplicate is a no-op.

```compact
ledger allowlist: Set<Bytes<32>>;

export circuit addAddress(addr: Bytes<32>): [] {
  allowlist.insert(disclose(addr));
}

export circuit isAllowed(addr: Bytes<32>): Boolean {
  return allowlist.member(addr);
}
```

Operations:

| Operation | Syntax | What it does |
|----------|--------|-----------|
| Insert | `set.insert(v)` | Add value, duplicate is no-op |
| Remove | `set.remove(v)` | Remove value |
| Check | `set.member(v)` | Returns `Boolean` |
| Empty check | `set.isEmpty()` | Returns `Boolean` |
| Size | `set.size()` | Returns `Uint<64>` |
| Reset | `set.resetToDefault()` | Clears the set |

Useful for allowlists, denylists, and membership tracking.

---

## Map

An unbounded key-value mapping. Values can be other ledger-state types (except `Kernel`).

```compact
ledger balances: Map<Bytes<32>, Uint<64>>;

export circuit setBalance(addr: Bytes<32>, amount: Uint<64>): [] {
  balances.insert(disclose(addr), disclose(amount));
}

export circuit getBalance(addr: Bytes<32>): Uint<64> {
  return balances.lookup(addr);
}
```

Operations:

| Operation | Syntax | What it does |
|----------|--------|-----------|
| Insert | `map.insert(k, v)` | Add or update key-value |
| Lookup | `map.lookup(k)` | Returns value (errors if missing) |
| Check | `map.member(k)` | Returns `Boolean` |
| Remove | `map.remove(k)` | Delete key-value |
| Empty check | `map.isEmpty()` | Returns `Boolean` |
| Size | `map.size()` | Returns `Uint<64>` |

### Nested Maps

```compact
ledger fld: Map<Boolean, Map<Field, Counter>>;

export circuit increment(b: Boolean, n: Field, k: Uint<16>): [] {
  fld.lookup(b).lookup(n) += disclose(k);
}
```

Rules for nesting:
- Nested values must be initialized before first use.
- The entire indirection chain must be used in a single expression.
- Only `Map` values can contain ledger-state types.
- `Kernel` cannot be nested.

---

## List

An unbounded ordered list. Elements are pushed and popped from the front.

```compact
ledger queue: List<Field>;

export circuit enqueue(v: Field): [] {
  queue.pushFront(disclose(v));
}

export circuit dequeue(): [] {
  queue.popFront();
}

export circuit peek(): Field {
  return queue.head().value;
}
```

Operations:

| Operation | Syntax | What it does |
|----------|--------|-----------|
| Push | `list.pushFront(v)` | Add to front |
| Pop | `list.popFront()` | Remove from front |
| Head | `list.head()` | Returns `Maybe<T>` |
| Empty check | `list.isEmpty()` | Returns `Boolean` |
| Length | `list.length()` | Returns `Uint<64>` |

`head()` returns `Maybe<T>`, check `.isSome` before accessing `.value`.

---

## MerkleTree

A bounded Merkle tree of depth `n` (2 ≤ n ≤ 32). Use when you need to prove membership against the **current** root.

```compact
ledger members: MerkleTree<8, Bytes<32>>;

export circuit addMember(leaf: Bytes<32>): [] {
  members.insert(disclose(leaf));
}

export circuit checkMembership(root: MerkleTreeDigest): Boolean {
  return members.checkRoot(root);
}
```

Operations:

| Operation | Syntax | What it does |
|----------|--------|-----------|
| Insert | `merkle.insert(v)` | Add leaf, update root |
| Check | `merkle.checkRoot(digest)` | Verify leaf against current root |
| Full check | `merkle.isFull()` | Returns `Boolean` |
| Root | `merkle.root()` | Read-only in TypeScript |

**What this enables:** A compact proof that a value is a member of the set, without revealing the value or the full set. The proof is a Merkle path, a sequence of sibling hashes from the leaf to the root.

**Use case:** Prove you know a secret in a set without revealing the secret.

---

## HistoricMerkleTree

Like `MerkleTree` but retains past roots. Use when you need to prove membership against **any past root**.

```compact
ledger commitments: HistoricMerkleTree<16, Bytes<32>>;

export circuit addCommitment(leaf: Bytes<32>): [] {
  commitments.insert(disclose(leaf));
}

export circuit verifyOldMembership(root: MerkleTreeDigest): Boolean {
  return commitments.checkRoot(root);  // accepts any past root
}
```

**What this enables:** Proving that a commitment existed at a past point in time. Useful for nullifier-based systems where you need to prevent double-spending.

---

## Kernel

A special ledger-state type that provides access to built-in ledger operations.

```compact
ledger kern: Kernel;

export circuit checkTime(deadline: Uint<64>): Boolean {
  return kern.blockTimeLessThan(deadline);
}

export circuit selfAddress(): ContractAddress {
  return kern.self();
}
```

Operations:

| Operation | What it does |
|-----------|-------------|
| `kern.self()` | Returns this contract's address |
| `kern.blockTimeLessThan(t)` | Check time against deadline |
| `kern.balance()` | Check native token balance |
| `kern.mintShielded(...)` | Mint shielded tokens |
| `kern.mintUnshielded(...)` | Mint unshielded tokens |
| `kern.checkpoint()` | Create a checkpoint for upgrades |

`Kernel` cannot be nested in a `Map`.

---

## Choosing the Right ADT

| Use case | ADT | Why |
|---------|-----|-----|
| Single mutable value | `Cell<T>` | Simple, direct access |
| Atomic counter (low contention) | `Counter` | Increment without read |
| Membership tracking | `Set<T>` | O(1) insert, check, remove |
| Per-key storage | `Map<K, V>` | Key-value with O(1) lookup |
| Nested per-key storage | `Map<K, Map<K2, V>>` | Two-level lookup |
| Ordered queue | `List<T>` | Push/pop from front |
| Current membership proof | `MerkleTree<n, T>` | Single-root membership |
| Historical membership proof | `HistoricMerkleTree<n, T>` | Multi-root membership |
| Block time, tokens, address | `Kernel` | Built-in operations |

---

## Common Mistakes

1. **Using `Map` when `Set` suffices.** If you only need membership check, use `Set`. `Map` has more overhead for the value storage.

2. **Forgetting that `List` pushes from front.** `pushFront` adds to the front, `popFront` removes from the front. This is a stack, not a queue, the name reflects the storage order, not the access pattern.

3. **Not initializing nested values.** Before accessing `map.lookup(k).lookup(k2)`, the inner `Map` for `k` must exist. The compiler requires the entire chain in one expression.

4. **Using `MerkleTree` when you need historical proofs.** `MerkleTree` only proves against the current root. If you need past roots, use `HistoricMerkleTree`.

5. **Nesting `Kernel` in `Map`.** Not allowed. `Kernel` is a special type with built-in operations, it cannot be nested.

---

## Comparison Layer

| ADT | Solidity equivalent | Note |
|-----|---------------------|-----|
| `Cell<T>` | `uint256` | Simple storage slot |
| `Counter` | N/A | Atomic increment |
| `Set<T>` | `mapping → bool` | Unique membership |
| `Map<K, V>` | `mapping(K ⇒ V)` | Key-value |
| `List<T>` | `uint256[]` | Ordered with overhead |
| `MerkleTree` | N/A | ZK-native, not Solidity-native |
| `Kernel` | built-in | Block time, tokens |

---

## Quick Recap

- **Counter**, Low contention, atomic increment without read.
- **Cell**, Default for single values. Read/write shorthand.
- **Set**, Membership tracking, unique values.
- **Map**, Key-value with nested ledger types.
- **List**, Ordered stack, push/pop from front.
- **MerkleTree**, Current-root membership proofs.
- **HistoricMerkleTree**, Any-past-root membership proofs.
- **Kernel**, Built-in operations (time, tokens, address). Cannot be nested.
- Choose based on access patterns. The ADT affects what your circuits can prove.

---

## Cross-Links

- **Previous:** [Data Types](./chapter-08.md)  Type system
- **Next:** [Standard Library](./chapter-10.md)  Hashing and token functions
- **See also:** [Ledger State](./chapter-04.md)  Commitment patterns
- **See also:** [Standard Library](./chapter-10.md)  Merkle tree functions
- **Examples:** [09.01 ADTs](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/09.01.adts.compact) · [09.02 Merkle Trees](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/09.02.merkle.compact) · [09.03 Kernel](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/09.03.kernel.compact)