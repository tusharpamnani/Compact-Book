# Circuits

This note explains circuits, the operational unit of Compact, and why they're fundamentally different from regular functions.

> **Docs:** [Circuit Definitions](https://docs.midnight.network/compact/reference/compact-reference#circuit-definitions)
> **Examples:** [05.01 Basic Circuits](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/05.01.circuits.compact) · [05.02 Generics](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/05.02.generics.compact) · [05.03 Control Flow](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/05.03.control-flow.compact)

---

## Intuition First

A circuit is not a function. A function takes inputs, executes statements in sequence, and returns an output. A circuit declares constraints on inputs that must hold true.

This distinction matters. When you write `return a + b`, you're not "computing" a + b, you're asserting that the output equals a + b. The proof proves this relationship held for the given inputs.

Circuits are the language of constraints. Functions are the language of computation. Compact uses constraints because that's what ZK proofs verify.

---

## Mental Model

**A circuit is a constraint system, not a procedure.**

| Functions | Circuits |
|-----------|---------|
| Execute statements in sequence | Declare relationships between inputs/outputs |
| Return values | Assert equalities between expressions |
| Can have side effects | No side effects (pure circuits) |
| Run at runtime | Compile to gates and constraints |
| Reveal inputs/outputs | Proves constraints satisfied without revealing inputs |

When you call a circuit from your DApp, you're not "running" it in the traditional sense. You're generating a proof that the declared constraints held for the actual inputs.

---

## Syntax

```compact
export pure circuit name<GenericParams>(param: Type, ...): ReturnType {
  // body
}
```

| Part | Meaning |
|------|--------|
| `export` | Callable from TypeScript (your DApp) |
| `pure` | Optional. Asserts no side effects (ledger reads/writes, witness calls) |
| `circuit` | The keyword, this is a constraint declaration |
| `GenericParams` | Optional type parameters |
| `param: Type` | Each parameter must have an explicit type |
| `ReturnType` | Must be explicitly declared |

---

## Simple Examples

```compact
circuit add(a: Uint<64>, b: Uint<64>): Uint<64> {
  return a + b;
}

export circuit get(): Uint<64> {
  assert(state == State.SET, "Value is not set");
  return value;
}
```

The `assert` statement is itself a constraint. The circuit fails if the condition is false, the proof won't verify.

---

## Pure vs Impure

A circuit is **pure** if it computes outputs from inputs only, no ledger reads, no ledger writes, no witness calls.

```compact
pure circuit hashPair(a: Field, b: Field): Field {
  return transientHash<[Field, Field]>([a, b]);
}
```

If you mark a circuit `pure` but it accidentally calls a witness, the compiler catches it. Pure circuits appear in the `PureCircuits` TypeScript type, they're the circuits that don't need state access.

**Why mark pure?** It documents intent and lets the compiler verify it. If you're using a circuit in a context where ledger state shouldn't change (like computing a hash for a commitment), `pure` ensures you didn't accidentally depend on ledger state.

---

## Parameters and Destructuring

```compact
circuit double(x: Field): Field { return x + x; }

// tuple destructuring
circuit sumPair([a, b]: [Field, Field]): Field { return a + b; }

// struct destructuring
struct Point { x: Uint<32>, y: Uint<32> }
circuit sumPoint({x, y}: Point): Uint<64> { return x + y; }

// rename a field
circuit useX({x: val}: Point): Uint<32> { return val; }
```

Destructuring works at the parameter level. This is syntactic sugar, it's the same as:

```compact
circuit sumPoint(p: Point): Uint<64> { return p.x + p.y; }
```

---

## Return Types

Use `[]` for circuits that return nothing (they only update state):

```compact
export circuit clear(): [] {
  state = State.UNSET;
}
```

`[]` means "no return value." The circuit still produces a proof, it proves the state was updated correctly.

---

## Local Bindings

```compact
circuit compute(x: Field): Field {
  const doubled = x + x;
  const result = doubled * 3;
  return result;
}

const [a, b] = pair;
const {x, y} = point;
```

Variables are immutable after binding. `const x = ...; x = ...;` is invalid. This isn't a style choice, it's enforced because circuits declare constraints, and reassignment would be ambiguous in a constraint system.

---

## Control Flow

```compact
assert(state == State.SET, "Value is not initialized");

if (condition) {
  // then
} else {
  // else
}
```

`if`/`else` works, but there's no early return from within a `for` body. This is because circuits need to declare all constraints, a `return` inside a loop would make the constraint structure conditional in a way the compiler can't handle.

---

## For Loops

Both forms are bounded at compile time:

```compact
// iterate over vector
for (const x of v) { }

// iterate over numeric range (0..N excludes N)
for (const i of 0..10) { }
```

`return` cannot be used inside a `for` body. Use `map` and `fold` for accumulation.

---

## map and fold

For transformation and accumulation without `return`:

```compact
circuit doubleAll(v: Vector<4, Field>): Vector<4, Field> {
  return map((x) => x + x, v);
}

circuit sumAll(v: Vector<4, Field>): Field {
  return fold((acc, x) => acc + x, 0, v);
}
```

`map` applies a function to each element. `fold` accumulates across elements. These are idiomatic in Compact because `for` loops can't use `return`.

---

## Generic Circuits

```compact
circuit identity<T>(x: T): T { return x; }

circuit firstOf<#N, T>(v: Vector<N, T>): T { return v[0]; }
```

Generic circuits must be specialized at the call site:

```compact
const x = identity<Field>(42);
const head = firstOf<4, Field>([1, 2, 3, 4]);
```

Generic circuits cannot be exported from the top level, they must be specialized before export.

---

## What Actually Happens Under the Hood

When you compile a circuit:

```
Compact code
        ↓
Compiler generates constraint system (ZKIR)
        ↓
Constraint system → arithmetic circuit (gates + wires)
        ↓
At runtime: inputs + proof keys → ZK proof
        ↓
Proof submitted → verified → state updated
```

The circuit declares constraints. The proof system converts those constraints into an arithmetic circuit. The proof proves the arithmetic circuit was satisfied.

**The key insight:** The circuit doesn't "execute" at verification time. The proof contains enough information for anyone to verify the constraints held without re-executing the computation.

---

## Common Mistakes

1. **Thinking circuits execute like functions.** Circuits declare constraints. The proof proves the constraints were satisfied. "Running" a circuit means generating a proof.

2. **Using `return` inside `for` loops.** Not allowed. The circuit structure must be fully determined at compile time. Use `map` and `fold` instead.

3. **Forgetting that variables are immutable.** `const x = 1; x = 2;` is invalid. Once bound, a value cannot change. Use `fold` or multiple variables if you need accumulation.

4. **Marking impure circuits `pure`.** If a circuit reads the ledger or calls a witness, it's impure. The compiler catches mismatches, but understanding why matters: `pure` is a promise about no side effects.

5. **Not using `assert` on witness outputs.** Witnesses are untrusted. The ZK proof proves the circuit's logic ran correctly, it doesn't prove the inputs were sensible. Always validate: `assert(balance >= amount, "Insufficient")`.

---

## Comparison Layer

| Concept | TypeScript functions | Rust | Solidity | Compact circuits |
|---------|-----------------|------|---------|-------------|
| Parameters | typed | typed | typed | typed |
| Return | explicit | explicit | explicit | explicit |
| Side effects | allowed | ownership | storage | **not in pure** |
| Recursion | allowed | allowed | allowed | **not allowed** |
| Unbounded loops | allowed | allowed | gas-limited | **not allowed** |
| Immutability | `const` | `let` | implicit | `const` |
| Execution | at runtime | at runtime | at runtime | at compile (proves at runtime) |

---

## Quick Recap

- A circuit is a constraint declaration, not a procedure.
- `return a + b` asserts output equals a + b. It doesn't "compute" it in sequence.
- Pure circuits have no side effects. Impure circuits read/write ledger state.
- Variables are immutable after binding. No reassignment.
- No `return` inside `for` loops. Use `map` and `fold`.
- Generic circuits must be specialized at the call site.
- The ZK proof proves constraints were satisfied, it doesn't reveal inputs.

---

## Cross-Links

- **Previous:** [Ledger State](./chapter-04.md)  Public state model
- **Next:** [Witnesses](./chapter-06.md)  How private inputs enter circuits
- **See also:** [What is Compact](./chapter-00.md)  Compilation overview
- **See also:** [Explicit Disclosure](./chapter-07.md)  Privacy boundary
- **Examples:** [05.01 Basic Circuits](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/05.01.circuits.compact) · [05.02 Generics](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/05.02.generics.compact) · [05.03 Control Flow](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/05.03.control-flow.compact)