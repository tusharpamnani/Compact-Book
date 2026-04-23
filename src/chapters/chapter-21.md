# Constructors

This note covers Compact constructors - the initialization logic that runs once when a contract is deployed.

> **Docs:** [Writing a Contract](https://docs.midnight.network/compact/reference/writing)

---

## Intuition First

A constructor runs exactly once: when you deploy the contract. Set up your initial state here.

---

## Basic Constructor

```compact
pragma language_version 0.22;

import CompactStandardLibrary;

export ledger owner: Bytes<32>;
export ledger count: Uint<64>;

constructor() {
  owner = disclose(callerAddress());
  count = 0;
}
```

### With Parameters

```compact
constructor(name: Bytes<32>, maxSupply: Uint<64>) {
  name = disclose(name);
  maxSupply = disclose(maxSupply);
  creator = disclose(callerAddress());
}
```

---

## What Can Go in a Constructor

- Setting initial values
- Creating empty collections
- Recording deployer

---

## What Cannot Go in a Constructor

- Circuit calls
- Witness calls
- Time-dependent logic

---

## Quick Recap

- Constructor runs once at deployment
- Set initial state, record deployer
- Parameters come from deployment transaction
- Cannot call circuits or witnesses

---

## Cross-Links

- **Previous:** [Explicit Disclosure](./chapter-11.md)
- **Next:** [Ledger ADTs](./chapter-12.md)
- **See also:** [Writing A Contract](./chapter-14.md)