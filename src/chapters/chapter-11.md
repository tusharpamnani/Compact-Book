# Explicit Disclosure

This note is the definitive guide to `disclose()`, what it means, how the compiler tracks witness data, and the common mistakes that trigger errors.

> **Docs:** [Explicit Disclosure](https://docs.midnight.network/compact/reference/explicit-disclosure)
> **Examples:** [07.01 disclose](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/07.01.disclose.compact)

---

## Intuition First

`disclose()` is not a cryptographic operation. It does not encrypt, hide, or transform data. It is a **compile-time annotation** that tells the compiler: "I am intentionally making this data public."

The compiler enforces this boundary. If witness data (from a `witness` callback) reaches the public ledger without `disclose()`, the compiler errors. This is the witness protection program, it prevents accidental privacy leaks.

Understanding `disclose()` means understanding that the compiler tracks data flow, not that there's magic happening at runtime.

---

## The Core Rule

A Compact program must explicitly declare its intention to disclose data that might be private before:

1. **Storing it in the public ledger**, `ledger x = disclose(witness())`
2. **Returning it from an exported circuit**, `return disclose(witness())`
3. **Passing it to another contract**, via `sendUnshielded`

**Privacy is the default.** `disclose()` is the explicit exception.

---

## What Counts as Witness Data

Witness data originates from:

- **Return values of `witness` function calls**, The secret key, balance, etc.
- **Arguments passed to exported circuits**, These come from the DApp, which may contain witness-derived data
- **Arguments passed to the contract constructor**, Same as above

Any value **derived** from witness data is also witness data. The taint follows the data everywhere.

---

## The `disclose()` Flow

![The disclose() Flow](../images/disclose_flow.png)

---

## `disclose()` Syntax

```compact
disclose(expr)
```

Wraps any expression. The compiler checks if `expr` contains witness data, if it does, the annotation is recorded and the disclosure is permitted.

```compact
// Basic: disclose a witness value
ledger balance: Uint<64>;
export circuit record(): [] {
  balance = disclose(getBalance());
}

// Array: disclose only the private element
const result = [publicValue, disclose(privateValue)];

// Function: disclose the return value
return disclose(helper(witnessData));
```

Place `disclose()` as close to the disclosure point as possible. This minimizes the scope of what you're declaring public.

---

## The Compiler Error

When you forget `disclose()`, the compiler gives you a detailed trace:

```
Exception: line 6 char 11:
  potential witness-value disclosure must be declared but is not:
    witness value potentially disclosed:
      the return value of witness getBalance at line 2 char 1
    nature of the disclosure:
      ledger operation might disclose the witness value
    via this path through the program:
      the right-hand side of = at line 6 char 11
```

This tells you:
1. Where the witness data came from (`getBalance`)
2. Where it tried to go (the ledger)
3. The exact path through the program

---

## Indirect Disclosure

You cannot hide witness data by passing it through arithmetic or helper circuits. The compiler tracks data flow through every operation.

```compact
// Compiler catches this
circuit obfuscate(x: Field): Field {
  return x + 73;  // output is still witness data
}

export circuit record(): [] {
  const s = getBalance() as Field;
  const x = obfuscate(s);
  balance = x as Bytes<32>;  // error
}
```

Even `x + 73` doesn't hide the witness data. The compiler knows the output depends on the input.

---

## Disclosure Via Return Values

Returning witness-derived data from an exported circuit is a disclosure:

```compact
// compiler error: balance flows to return
export circuit balanceExceeds(n: Uint<64>): Boolean {
  return getBalance() > n;  // comparison result depends on witness data
}

// correct: declared disclosure
export circuit balanceExceeds(n: Uint<64>): Boolean {
  return disclose(getBalance()) > n;
}
```

Even a comparison result counts. If the output can be determined by witness data, it's a disclosure.

---

## Where to Place `disclose()`

**Place it as close to the disclosure point as possible:**

```compact
// Wrong: declares more than necessary
export circuit process(data: PrivateData): [] {
  const result = compute(disclose(data));  // too early
  ledger = result;
}

// Right: declares only what's needed
export circuit process(data: PrivateData): [] {
  const result = compute(data);
  ledger = disclose(result);  // only the final output
}
```

The earlier you disclose, the more you declare public. Wait until the last possible moment.

---

## Standard Library Exceptions

Some functions handle witness data without explicit disclosure:

| Function | Witness-tainted? | Why |
|----------|-----------------|-----|
| `transientCommit(e)` | **No** | Random nonce provides sufficient hiding |
| `transientHash(e)` | Yes | Bare hash may not hide input |

```compact
// no disclose() needed: transientCommit's nonce hides the value
ledger commitment: Field;
export circuit commit(v: Field): [] {
  const nonce = freshNonce();
  commitment = transientCommit(v, nonce);  // ok
}

// disclose() needed: transientHash doesn't hide
ledger hashed: Field;
export circuit storeHash(v: Field): [] {
  hashed = disclose(transientHash(v));  // required
}
```

The nonce in `transientCommit` provides enough randomness that even someone who knows the value can't determine the commitment without the nonce.

---

## The Two-World Model, Revisited

```
┌─────────────────────────────────────────────────────────────┐
│                    PRIVATE WORLD                           │
│                                                             │
│  witness getBalance(): Uint<64>;                          │
│  returns: 1000  (never on chain)                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼ (disclose())
                            │
┌─────────────────────────────────────────────────────────────┐
│                    PUBLIC WORLD                          │
│                                                             │
│  export ledger balance: Uint<64>;                        │
│  balance = disclose(getBalance());                       │
│  stored: 1000  (public, declared)                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Without `disclose()`, the compiler draws this line and prevents crossing.

---

## Common Mistakes

1. **Thinking `disclose()` encrypts.** It doesn't. It's a compile-time annotation. There's no runtime cost and no cryptographic transformation. If you disclose something, it's public.

2. **Forgetting indirect disclosure.** Passing witness data through `obfuscate(x) = x + 73` doesn't hide it. The compiler tracks data flow through every operation.

3. **Not disclosing comparison results.** `return getBalance() > n` is a disclosure, the comparison reveals information about the balance. Use `disclose()`.

4. **Disclosing too early.** Placing `disclose()` at the start of a function declares everything derived from that value as public. Place it at the last possible moment.

5. **Assuming `transientHash` hides witness data.** It doesn't. Use `transientCommit` if you need hiding without disclosure. Or use `disclose(transientHash(...))`.

---

## Comparison Layer

| Concept | Solidity | TypeScript | Compact |
|---------|---------|-----------|---------|
| Private data | `private` (still on chain) | `private` fields | `witness` |
| Privacy mechanism | encryption (optional) | memory isolation | ZK proofs |
| Disclosure | explicit in code | code logic | `disclose()` annotation |
| Enforcement | contract code | convention | **compiler** |

The key difference: in Solidity, privacy is a convention. In Compact, it's enforced by the compiler.

---

## Quick Reference

| Scenario | Require `disclose()`? |
|----------|---------------------|
| Store witness value in ledger | Yes |
| Return witness value from exported circuit | Yes |
| Return comparison result | Yes |
| Use witness data inside a circuit (no ledger access) | No |
| Store `transientCommit(witnessValue)` in ledger | **No** (nonce hides) |
| Store `transientHash(witnessValue)` in ledger | Yes |
| Store `persistentCommit(witnessValue)` in ledger | **No** (nonce hides) |
| Store `persistentHash(witnessValue)` in ledger | Yes |

---

## Cross-Links

- **Previous:** [Witnesses](./chapter-06.md)  Private input mechanism
- **Next:** [Data Types](./chapter-08.md)  Type system
- **See also:** [Ledger State](./chapter-04.md)  The two-world model
- **See also:** [Standard Library](./chapter-10.md)  Hash functions
- **Examples:** [07.01 disclose](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/07.01.disclose.compact)