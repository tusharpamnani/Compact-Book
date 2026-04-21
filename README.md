# Compact Notes

A structured, example-driven reference for learning **Compact** — the smart contract language of the [Midnight Network](https://midnight.network).

Think of this as the Rust Book or Anchor Cookbook, but for Compact. Every note is grounded strictly in the [official Midnight documentation](https://docs.midnight.network). No guesswork, no invented patterns.

---

## What is Midnight?

Midnight is a data protection blockchain. It maintains two parallel states:

- **Public state** — on-chain, visible to all participants (proofs, contract code, public data)
- **Private state** — stored locally by users, never exposed to the network (sensitive data, personal information)

The bridge between the two is zero-knowledge cryptography (zk-SNARKs). Computations happen locally on private data; only the proof of correctness goes on-chain. Validators verify the proof without ever seeing the underlying inputs.

## What is Compact?

Compact is Midnight's domain-specific smart contract language. It is based on TypeScript and compiles down to zero-knowledge circuits. You write familiar, structured code — the compiler handles the cryptography.

> "Instead of requiring cryptographic expertise, developers write familiar code that automatically compiles to zero-knowledge circuits." — [docs.midnight.network](https://docs.midnight.network/what-is-midnight)

---

## Who is this for?

- Developers coming from TypeScript/JavaScript who want to build on Midnight
- Solidity or Rust developers exploring ZK-native smart contracts
- Anyone who learns better from structured notes + code examples than raw documentation

---

## Table of Contents

| # | Topic | Description |
|---|-------|-------------|
| 00 | [What is Compact](./00.%20What%20is%20Compact.md) | Language overview, what it compiles to, and where it fits in the Midnight stack |
| 01 | [Why Compact](./01.%20Why%20Compact.md) | Design goals, tradeoffs, and how it compares to writing ZK circuits manually |
| 02 | [Setting Up the Compiler](./02.%20Setting%20Up%20the%20Compiler.md) | Installing the Compact compiler and toolchain |
| 03 | [Writing a Contract](./03.%20Writing%20a%20Contract.md) | Structure of a Compact contract — ledger, circuits, witnesses |
| 04 | [Ledger State](./04.%20Ledger%20State.md) | Public vs private state, how ledger declarations work |
| 05 | [Circuits](./05.%20Circuits.md) | What circuits are, how they map to ZK proofs, circuit syntax |
| 06 | [Witnesses](./06.%20Witnesses.md) | The witness block — providing private inputs to circuits |
| 07 | [Explicit Disclosure](./07.%20Explicit%20Disclosure.md) | How and when to use `disclose()`, selective revelation of private data |
| 08 | [Data Types](./08.%20Data%20Types.md) | Primitive types: `Field`, `Boolean`, `Uint`, `Bytes`, `CurvePoint` |
| 09 | [Ledger ADTs](./09.%20Ledger%20ADTs.md) | `Map`, `Set`, `MerkleTree` — how Compact manages on-chain collections |
| 10 | [Standard Library](./10.%20Standard%20Library.md) | Built-in functions available in Compact contracts |
| 11 | [Modules and Imports](./11.%20Modules%20and%20Imports.md) | How Compact organizes code across files and modules |
| 12 | [Compact Grammar](./12.%20Compact%20Grammar.md) | Formal grammar reference — syntax rules for the language |
| 13 | [Keywords Reference](./13.%20Keywords%20Reference.md) | All Compact keywords and what they do |
| 14 | [Testing and Debugging](./14.%20Testing%20and%20Debugging.md) | How to test Compact contracts, common errors and how to read them |
| 15 | [Security Best Practices](./15.%20Security%20Best%20Practices.md) | Patterns to avoid, known footguns, and how to write safe contracts |

---

## How to Use This Repo

Each note is a self-contained `.md` file. Where code is involved, there is a companion `.compact` file with the same number prefix. Read the note first, then study the code file.

The notes follow the official Midnight docs strictly. Where the docs are the source, a reference link is included. Nothing in these notes is invented.

---

## Resources

- [Midnight Developer Docs](https://docs.midnight.network)
- [Compact Language Reference](https://docs.midnight.network/compact)
- [Official Examples (awesome-midnight-dapps)](https://github.com/midnightntwrk/midnight-awesome-dapps)
- [Midnight Discord](https://discord.com/invite/midnightnetwork)
- [Midnight Forum](https://forum.midnight.network)

---

## Contributing

If something is wrong or outdated, open an issue or PR. All corrections must be traceable to the official docs.