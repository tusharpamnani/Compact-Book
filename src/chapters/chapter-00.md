# What is Compact?

**Compact** is Midnight's smart contract language, a TypeScript-like DSL that compiles to zero-knowledge circuits.

The key word is *compiles*. You're not writing circuits directly. You're writing TypeScript-looking code, and the compiler produces the cryptographic machinery. This is the core value proposition: you get ZK correctness guarantees without learning ZK circuit design.

---

## Intuition First

Compact looks like TypeScript because it was designed that way. But it is emphatically *not* TypeScript running somewhere, it is a constrained language where every program maps to a finite, deterministic circuit.

The constraints exist for a reason. ZK proofs require finite computation. If Compact let you write unbounded loops or recursion, the compiler couldn't produce a circuit. So instead of fighting these constraints, accept them as the mechanism that makes the magic work.

When you write a circuit in Compact, you're describing constraints on inputs. When you call that circuit from your DApp, the proof system generates a zero-knowledge proof that the constraints were satisfied, without revealing the inputs. This is why Compact programs can operate on private data while producing public verifiability.

---

## Mental Model

**A Compact program is a constraint declaration, not a procedure.**

In traditional programming, a function takes inputs and returns outputs by executing statements in sequence. In Compact, a circuit declares relationships between inputs and outputs that must hold true. The proof proves the relationships held, it doesn't "execute" in the normal sense.

| Traditional code | Compact circuit |
|------------------|-----------------|
| Execution happens | Constraints are declared |
| Inputs → statements → outputs | Inputs, outputs, and their relationships are asserted |
| Runs on a machine | Compiles to a circuit |
| Reveals everything | Proves correctness without revealing inputs |

**The boundedness principle:** Every type has a fixed size at compile time. Every loop has a known bound. Recursion is not allowed. This isn't a limitation, it's the mechanism that makes compilation to circuits possible.

---

## Where Compact Fits in the Midnight Stack

A Midnight smart contract has three parts:

1. **Public ledger component**, Replicated on-chain state, visible to all (proofs, contract code, public data)
2. **Zero-knowledge circuit component**, Proves correct execution without revealing private inputs
3. **Off-chain component**, Arbitrary TypeScript code that drives the contract (wallet integration, proof generation)

Compact handles parts 1 and 2. You write Compact; the compiler generates the ZK circuits and TypeScript bindings. Your DApp uses the bindings to call circuits and submit proofs.

---

## The Two-World Model

![Two-World Model](../images/two_worlds.png)

**Privacy is the default.** Private data stays local. Only the proof goes on-chain.

---

## What Compact Looks Like

Compact's syntax is deliberately close to TypeScript:

```compact
pragma language_version 0.22;

import CompactStandardLibrary;

enum State { UNSET, SET }

export ledger value: Uint<64>;
export ledger state: State;

export circuit get(): Uint<64> {
  assert(state == State.SET, "Value not set");
  return value;
}
```

If you've written TypeScript, this looks familiar. The differences are what make it ZK-suitable:

- **No `any` type.** Every expression has a known type at compile time.
- **Numeric types are unsigned only.** Either bounded (`Uint<0..n>`) or sized (`Uint<n>`) or `Field`.
- **No recursion.** Circuits cannot call themselves.
- **Bounded loops only.** Loop bounds must be compile-time constants.
- **Privacy boundaries are enforced.** Witness data must be explicitly disclosed before flowing into public state.

---

## Why These Constraints Exist

Compact is intentionally not Turing-complete. This is not a weakness, it is the design:

| Constraint | Why it exists |
|-----------|-------------|
| No recursion | Circuit depth must be finite |
| Bounded loops only | The circuit size is determined at compile time |
| Fixed type sizes | Memory layout in circuits is fixed |
| No `any` | The compiler must track data flow to enforce privacy |
| Unsigned only | Field arithmetic is cleaner; sign is handled differently |

ZK proofs require finite circuits. Compact enforces finiteness at the language level, so compilation always succeeds for type-correct programs.

---

## What the Compiler Produces

![Compiler Output](../images/compiler_output.png)

When you compile a contract, the output maps to the visual above:

| Output directory | What it is |
|------------------|------------|
| `contract/index.js` | TypeScript runtime, your DApp calls this |
| `contract/index.d.ts` | Type type definitions |
| `compiler/contract-info.json` | Metadata (circuit list, witness list) |
| `zkir/*.zkir` | ZK intermediate representation, the circuit definition |
| `keys/*.prover` | Proving key, generates proofs |
| `keys/*.verifier` | Verification key, validates proofs |

### The Compiled JavaScript

The `index.js` is a runtime class with methods for each exported circuit:

```typescript
import { Contract, ledger } from './contract';

const witnesses = { callerAddress };
const contract = new Contract(witnesses);

const result = await contract.circuits.mint(context, metadataHash);

const state = ledger(chargedState);
console.log(state.totalSupply);
```

### TypeScript Definitions

The `.d.ts` exports typed interfaces:

```typescript
export type Witnesses<PS> = {
  callerAddress(context: WitnessContext): [PS, Uint8Array];
}

export type ImpureCircuits<PS> = {
  mint(context: CircuitContext, metadataHash: Uint8Array): CircuitResults<PS, []>;
  transfer(
    context: CircuitContext,
    tokenId: bigint,
    newOwner: Uint8Array,
    tokenMetaHash: Uint8Array
  ): CircuitResults<PS, []>;
}

export type Ledger = {
  readonly totalSupply: bigint;
  readonly nextTokenId: bigint;
  tokenCommitments: {
    isEmpty(): boolean;
    size(): bigint;
    member(key: bigint): boolean;
    lookup(key: bigint): Uint8Array;
  };
}

export class Contract {
  witnesses: Witnesses;
  circuits: Circuits;
  impureCircuits: ImpureCircuits;
  constructor(witnesses: Witnesses);
  initialState(context: ConstructorContext): ConstructorResult;
}
```

### The ZKIR (Zero-Knowledge Intermediate Representation)

The `.zkir` files are the circuit definitions the ZK proof system consumes. You won't read these normally, but they're the output that matters:

```json
{
  "version": { "major": 2, "minor": 0 },
  "do_communications_commitment": true,
  "num_inputs": 2,
  "instructions": [
    { "op": "constrain_bits", "var": 0, "bits": 8 },
    { "op": "private_input", "guard": null },
    { "op": "persistent_hash", ... },
    { "op": "add", "a": 20, "b": 2 }
  ]
}
```

---

## The Five Program Elements

| Element | Keyword | Purpose |
|---------|---------|---------|
| Public state | `export ledger` | Declares on-chain, publicly readable state |
| Private inputs | `witness` | Declares callbacks for private data (body in TypeScript) |
| Logic | `circuit` | Functions that compile to ZK circuits |
| Initialization | `constructor` | Runs once on contract deployment |
| Organization | `module` / `import` / `type` | Namespace and type management |

---

## Quick Recap

- Compact is TypeScript-like code that compiles to zero-knowledge circuits.
- Every program must be finite, no recursion, bounded loops only, fixed type sizes.
- A circuit declares constraints, not procedures. The proof proves the constraints held.
- The compiler produces TypeScript bindings (your DApp uses these) and ZKIR (the proof system uses this).
- Privacy is the default: private data stays local; only the proof goes on-chain.
- `export ledger` is public. `witness` data is private. `disclose()` is the boundary.
- `assert` is your only runtime guard, use it to validate witness outputs.

---

## Common Mistakes

1. **Thinking circuits are like functions.** Circuits declare constraints, not steps. The proof proves the constraints held, it doesn't "execute" in sequence.

2. **Assuming unbounded loops work.** Loop bounds must be compile-time constants. `for (let i = 0; i < n; i++)` fails if `n` isn't known at compile time.

3. **Treating Compact as TypeScript.** No `any`, no dynamic typing, no recursion. The type system is strict by design.

4. **Forgetting the privacy boundary.** Private data (witnesses) cannot flow into public state without `disclose()`. The compiler catches this, but understanding why is important.

5. **Thinking `disclose()` encrypts.** It doesn't. It's a compile-time annotation that marks intentional disclosure. There's no runtime cost and no cryptographic transformation.

---

## Comparison Layer

| Concept | TypeScript | Solidity | Rust | Compact |
|---------|-----------|----------|------|---------|
| Private data | `private` keyword (convention) | Private variables (still on-chain) | Ownership model | Witnesses stay local |
| Public data | Database or API | Storage variables | `pub` fields | `export ledger` |
| Functions | Regular functions | `public`/`external` | `pub fn` | `export circuit` |
| Type safety | Optional (TypeScript) or dynamic (JS) | Loosely typed | Strong | Strong, enforced |
| Loops | Unbounded | Unbounded (gas-limited) | Unbounded | Bounded only |
| Recursion | Allowed | Allowed (stack limits) | Allowed | **Not allowed** |

---

## Cross-Links

- **Next:** [Why Compact](./chapter-01.md), Design goals and tradeoffs
- **See also:** [Circuits](./chapter-05.md), How circuits actually work under the hood
- **See also:** [Witnesses](./chapter-06.md), How private data enters circuits