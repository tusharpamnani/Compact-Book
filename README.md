# Compact Book

> The resource smart developers recommend when someone asks how to learn Compact properly.

A structured, example-driven guide for understanding **Compact**, the domain-specific language that compiles to zero-knowledge circuits on the Midnight blockchain. Built for developers who learn through code, mental models, and working examples, not documentation dumps.

Inspired by the [Rust-Notes](https://github.com/tusharpamnani/rust-notes) repo by [Tushar Pamnani](https://github.com/tusharpamnani).

---

## What is Midnight?

Midnight is a data-protection blockchain with **two parallel worlds**:

| World | Where | Who sees it | Contains |
|-------|-------|-----------|---------|
| **Public** | On-chain | Everyone | Proofs, contract code, public data |
| **Private** | Local storage | Only the owner | Sensitive data, secrets |

The bridge between them is zero-knowledge cryptography. Computations happen locally on private data; only a proof of correct execution goes on-chain. Validators verify the proof without ever seeing the inputs.

**This is the key insight:** Midnight doesn't hide computation, it proves computation happened correctly without revealing what you computed on.

---

## What is Compact?

Compact is Midnight's smart contract language. It looks like TypeScript, but every program compiles to a finite zero-knowledge circuit.

```
You write Compact (TypeScript-like)
        ↓
Compiler produces ZK circuits + TypeScript bindings
        ↓
Your DApp calls circuits → generates proof → submits to chain
```

The compiler handles all the cryptographic machinery. You write business logic.

---

## Who is this for?

| Background | Why this works for you |
|-----------|----------------------|
| **TypeScript developer** | Syntax is familiar. Mental models are different, but you already understand types and functions. |
| **Solidity developer** | You understand state and contracts. The privacy model is the new layer. |
| **Rust developer** | You're comfortable with strong types, ownership, and bounded computation. Compact will feel natural. |
| **ZK newcomer** | You don't need to understand circuits, you write code that compiles to them. |

You should be comfortable with at least one typed language before starting. Compact builds on that.

---

## Learning Path

Start at the top. Each note assumes you understood the previous one.

| # | Topic | What you learn | Level |
|---|-------|------------|---------|
| 00 | [What is Compact](./00.%20What%20is%20Compact.md) | What Compact is, what it compiles to, why boundedness matters | 🟢 Beginner |
| 01 | [Why Compact](./01.%20Why%20Compact.md) | Why this design exists, what problems it solves | 🟢 Beginner |
| 02 | [Setting Up the Compiler](./02.%20Setting%20up%20the%20compiler.md) | Install the toolchain, compile your first contract | 🟢 Beginner |
| 03 | [Writing a Contract](./03.%20Writing%20A%20Contract.md) | The four pieces of every contract | 🟡 Intermediate |
| 04 | [Ledger State](./04.%20Ledger%20State.md) | Public vs private state, the disclose() boundary | 🟡 Intermediate |
| 05 | [Circuits](./05.%20Circuits.md) | How circuits work, why they're not functions | 🟡 Intermediate |
| 06 | [Witnesses](./06.%20Witnesses.md) | How private inputs enter circuits without touching the chain | 🟡 Intermediate |
| 07 | [Explicit Disclosure](./07.%20Explicit%20Disclosure.md) | The witness protection program, common mistakes | 🔴 Advanced |
| 08 | [Data Types](./08.%20Data%20Types.md) | The complete type system | 🟡 Intermediate |
| 09 | [Ledger ADTs](./09.%20Ledger%20ADTs.md) | Map, Set, MerkleTree, choosing the right state structure | 🔴 Advanced |
| 10 | [Standard Library](./10.%20Standard%20Library.md) | Hashing, tokens, merkle trees | 🔴 Advanced |
| 11 | [Modules and Imports](./11.%20Modules%20and%20Imports.md) | Organizing code across files | 🟡 Intermediate |
| 12 | [Example Projects](./12.%20Example%20Projects.md) | Full working contracts | 🟢 Beginner |

**Reading order:** 00 → 01 → 02 → 03 → 12 (example projects) → then 04–11 as needed.

---

## What Makes These Notes Different

Most documentation explains what. These notes explain *why* and *what it actually means*.

Every note follows this structure:

1. **Intuition First**, The 30-second version before the details
2. **Mental Model**, How to think about it (precise, not vague)
3. **Why It Exists**, The problem it solves
4. **Core Example**, One clean example, then extended
5. **Under the Hood**, What actually happens during compilation
6. **Common Mistakes**, What you WILL get wrong (and why)
7. **Comparison Layer**, How it maps to Rust, Solidity, TypeScript
8. **Practical Usage**, Where it appears in real contracts
9. **Quick Recap**, Memory anchors

Cross-links connect concepts. Micro-challenges test your understanding.

---

## Core Mental Models to Build First

Before you write any code, understand these three:

**1. A circuit is a constraint system, not a function.**
It doesn't "run" in the normal sense. It declares constraints on inputs. The proof proves those constraints were satisfied.

**2. The ledger is public. Private data stays local.**
`export ledger` fields are on-chain and readable by everyone. Private data lives in witnesses and never touches the chain, only a proof that you operated on it correctly.

**3. `disclose()` is a compile-time annotation, not encryption.**
It tells the compiler "I am intentionally making this public." The compiler prevents accidental disclosure. There's no runtime cost and no magic.

---

## Quick Reference: The Five Program Elements

Every Compact contract is built from these:

| Element | Keyword | Purpose |
|---------|---------|---------|
| Public state | `export ledger` | On-chain, readable by all |
| Private inputs | `witness` | Callbacks the DApp provides |
| Logic | `circuit` | Compiles to ZK circuits |
| Initialization | `constructor` | Runs once on deploy |
| Organization | `module` / `import` | Namespace and file management |

---

## Resources

- [Midnight Developer Docs](https://docs.midnight.network), Official reference
- [Compact Language Reference](https://docs.midnight.network/compact), Spec details
- [Official Examples](https://github.com/midnightntwrk/midnight-awesome-dapps), Working projects
- [Midnight Discord](https://discord.com/invite/midnightnetwork), Community

---

## Contributing

Found a misconception that should be addressed? A better analogy? A mistake?

Open an issue or PR. This repo improves through developer feedback.