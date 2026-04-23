# Witnesses

This note explains witnesses, the mechanism that brings private data into circuits without ever touching the chain.

> **Docs:** [Declaring Witnesses](https://docs.midnight.network/compact/reference/compact-reference#declaring-witnesses-for-private-state-management)
> **Examples:** [06.01 Witnesses](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/06.01.witnesses.compact)

---

## Intuition First

A witness is a callback function. You declare its type in Compact, but the body is provided by your TypeScript DApp at runtime. When a circuit calls a witness, it runs locally on the user's device, the value it returns never goes on-chain. Instead, a ZK proof proves the circuit executed correctly given that value.

The name "witness" comes from ZK literature. In a proof, the witness is the secret data that proves a statement is true, without revealing what that data is. In Compact, witnesses are exactly that: secret inputs that prove the circuit ran correctly.

---

## Mental Model

**Witnesses are private inputs, not parameters.**

| Parameters (to circuits) | Witnesses |
|--------------------------|----------|
| Passed explicitly when calling | Provided by DApp at runtime |
| Visible in the proof inputs | Stay local, never on chain |
| Public (anyone can see them) | Private (only the caller knows) |
| Compiler enforces type | DApp provides implementation |

When you call a circuit, you pass public parameters. The circuit can also call witnesses internally. The witness returns a value, and the circuit uses it, but only the proof goes on-chain.

---

## The Flow

![Witness + Circuit Flow](../images/witness_circuit_flow.png)

---

## Declaring a Witness

```compact
witness secretKey(): Bytes<32>;
witness getBalance(addr: Bytes<32>): Uint<64>;
witness userNonce(): Field;
witness getItem<T>(index: Uint<32>): T;  // generic
```

Witness declarations have no body. The body is provided by your TypeScript DApp.

---

## Calling a Witness

```compact
export circuit clear(): [] {
  const sk = secretKey();           // call witness: returns private data
  const pk = publicKey(round, sk); // compute with it: still private
  assert(authority == pk, "Not authorized");
  state = State.UNSET;
  round.increment(1);
}
```

The witness call happens locally. The ZK proof proves the computation was correct, without revealing `sk`.

---

## The Compiler Tracks Witness Data

The compiler tracks witness data through every operation, arithmetic, type conversions, struct construction, function calls. Once data comes from a witness, it's "tainted", the compiler knows it's private.

```compact
export circuit example(): [] {
  const s = getSecret();
  const doubled = s + s;              // still witness data
  const converted = s as Uint<64>;    // still witness data
  ledger = doubled;                  // compiler error: undeclared disclosure
}
```

This is the **witness protection program**, the compiler prevents accidental disclosure of private data.

---

## When Disclosure Is Required

When witness data needs to flow into the public ledger, wrap it in `disclose()`:

```compact
// Without: compiler error
export circuit record(): [] {
  balance = getBalance();  // error: witness data going to ledger
}

// With: compiles
export circuit record(): [] {
  balance = disclose(getBalance());  // ok: declared
}
```

`disclose()` does not encrypt. It's a compile-time annotation that says "I'm intentionally making this public."

---

## The Compiler Error

When you forget `disclose()`, the compiler tells you exactly where the witness data came from:

```
Exception: line 6 char 11:
  potential witness-value disclosure must be declared but is not:
    witness value potentially disclosed:
      the return value of witness getBalance at line 2 char 1
    nature of the disclosure:
      ledger operation might disclose the witness value
```

This trace tells you:
1. Where the witness data originated (`getBalance`)
2. Where it tried to flow (the ledger assignment)

---

## Indirect Disclosure

The compiler catches disclosure even when witness data travels through helper circuits:

```compact
circuit obfuscate(x: Field): Field {
  return x + 73;  // output is still witness data
}

export circuit record(): [] {
  const s = getBalance() as Field;
  const x = obfuscate(s);
  balance = x as Bytes<32>;  // compiler catches this
}
```

The compiler's abstract interpreter follows witness taint through every operation. You cannot hide witness data by passing it through arithmetic, structs, or helper functions.

---

## Place Disclosure As Close As Possible

```compact
// Bad: declares broader scope
export circuit process(data: PrivateData): [] {
  const result = compute(disclose(data));  // discloses too much
  ledger = result;
}

// Good: discloses at the boundary
export circuit process(data: PrivateData): [] {
  const result = compute(data);
  ledger = disclose(result);  // discloses only what's needed
}
```

Place `disclose()` as close to the disclosure point as possible. This minimizes what you're declaring as public.

---

## Standard Library Exceptions

Some functions can handle witness data without explicit disclosure:

| Function | Witness-tainted? | Why |
|----------|-----------------|-----|
| `transientCommit(e)` | **No** | Random nonce provides sufficient hiding |
| `transientHash(e)` | Yes | Bare hash may not hide input |

```compact
// no disclose() needed
ledger commitment: Field;
export circuit commit(v: Field): [] {
  const nonce = freshNonce();
  commitment = transientCommit(v, nonce);  // nonce hides the value
}
```

The nonce provides enough randomness that even knowing the value doesn't help. This is why `transientCommit` doesn't require `disclose()`, but `transientHash` does.

---

## Critical: Witness Results Are Untrusted

> **Do not assume in your contract that the code of any `witness` function is the code that you wrote.** Any DApp may provide any implementation it wants. Results should be treated as untrusted input.

The ZK proof guarantees the circuit's logic ran correctly, given whatever inputs witnesses returned. It does not guarantee witnesses returned sensible values.

Your contract must validate witness outputs:

```compact
// WRONG: trust the witness
export circuit transfer(to: Bytes<32>, amount: Uint<64>): [] {
  const balance = getBalance();  // untrusted!
  balances[to] += amount;
}

// RIGHT: validate first
export circuit transfer(to: Bytes<32>, amount: Uint<64>): [] {
  const balance = getBalance();
  assert(balance >= amount, "Insufficient balance");  // validate!
  balances[to] += amount;
}
```

---

## Comparison Layer

| Concept | Solidity | TypeScript | Compact |
|---------|---------|-----------|---------|
| Private input | `private` variables | class fields | `witness` |
| Secret data | on-chain (encrypted) | in memory | stays local |
| Proving computation | N/A | N/A | via circuit |
| Trust model | contract code | app logic | witness is untrusted |

---

## Quick Recap

- Witnesses are callback functions, declared in Compact, implemented in TypeScript.
- They run locally on the user's device. The value never goes on-chain.
- Only the ZK proof goes on-chain, it proves the circuit ran correctly given the witness inputs.
- The compiler tracks witness data through every operation.
- If witness data reaches the ledger without `disclose()`, the compiler errors.
- `transientCommit` is an exception, the random nonce provides hiding without `disclose()`.
- **Always validate witness outputs.** The proof proves correct logic, not sensible inputs.

---

## Cross-Links

- **Previous:** [Circuits](./chapter-05.md)  Constraint declarations
- **Next:** [Explicit Disclosure](./chapter-07.md)  Deep dive on disclose()
- **See also:** [Ledger State](./chapter-04.md)  Public vs private
- **See also:** [Writing a Contract](./chapter-03.md)  Full example
- **Examples:** [06.01 Witnesses](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/06.01.witnesses.compact)