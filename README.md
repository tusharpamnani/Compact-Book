# Compact Notes

A structured, example-driven reference for learning **Compact**, the smart contract language of the [Midnight Network](https://midnight.network).

Inspired by the [Rust-Notes](https://github.com/tusharpamnani/rust-notes) repo by [Tushar Pamnani](https://github.com/tusharpamnani).

## What is Midnight?

Midnight is a data protection blockchain with two parallel states:

- **Public state**: on-chain, visible to all (proofs, contract code, public data)
- **Private state**: stored locally, never exposed (sensitive data)

The bridge is zero-knowledge cryptography. Computations happen locally; only the proof goes on-chain.

## What is Compact?

Compact is Midnight's domain-specific smart contract language. Based on TypeScript. Compiles to zero-knowledge circuits.

> "Instead of requiring cryptographic expertise, developers write familiar code that automatically compiles to zero-knowledge circuits." - Midnight Docs

## Who is this for?

- TypeScript/JavaScript developers building on Midnight
- Solidity or Rust developers exploring ZK-native smart contracts
- Anyone who learns better from code examples than raw documentation

## Table of Contents

| # | Topic | Description |
|---|-------|-------------|
| 00 | [What is Compact](./00.%20What%20is%20Compact.md) | Language overview and what it compiles to |
| 01 | [Why Compact](./01.%20Why%20Compact.md) | Design goals and tradeoffs |
| 02 | [Setting Up the Compiler](./02.%20Setting%20up%20the%20compiler.md) | Installing the toolchain |
| 03 | [Writing a Contract](./03.%20Writing%20A%20Contract.md) | Structure of a Compact contract |
| 04 | [Ledger State](./04.%20Ledger%20State.md) | Public vs private state |
| 05 | [Circuits](./05.%20Circuits.md) | What circuits are and how they work |
| 06 | [Witnesses](./06.%20Witnesses.md) | Providing private inputs to circuits |
| 07 | [Explicit Disclosure](./07.%20Explicit%20Disclosure.md) | How and when to use `disclose()` |
| 08 | [Data Types](./08.%20Data%20Types.md) | Primitive and program-defined types |
| 09 | [Ledger ADTs](./09.%20Ledger%20ADTs.md) | `Map`, `Set`, `MerkleTree`, etc. |
| 10 | [Standard Library](./10.%20Standard%20Library.md) | Built-in functions and types |
| 11 | [Modules and Imports](./11.%20Modules%20and%20Imports.md) | Code organization |
| 12 | [Example Projects](./12.%20Example%20Projects.md) | Working contract examples |

## How to Use This Repo

Each note is a self-contained `.md` file. Notes follow the official Midnight documentation strictly.

## Resources

- [Midnight Developer Docs](https://docs.midnight.network)
- [Compact Language Reference](https://docs.midnight.network/compact)
- [Official Examples](https://github.com/midnightntwrk/midnight-awesome-dapps)
- [Midnight Discord](https://discord.com/invite/midnightnetwork)