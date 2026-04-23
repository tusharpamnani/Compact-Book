# TypeScript Integration

This note covers how to use your compiled Compact contract from TypeScript, implement witnesses, and interact with the Midnight SDK.

> **Docs:** [Writing a Contract](https://docs.midnight.network/compact/reference/writing) · [SDK](https://docs.midnight.network/sdks)

---

## Intuition First

When you compile a Compact contract, you get TypeScript output. This output is your DApp's interface to the contract.

Three things happen on the TypeScript side:

1. **Witness implementations**, You provide the callback bodies for witnesses
2. **Circuit calls**, You call circuits through the generated API
3. **State reads**, You read ledger state to display to users

---

## The Compiled Output

After compiling, you get:

```
contracts/managed/myContract/
├── contract/
│   ├── index.js          # Main API
│   ├── index.d.ts      # Type definitions
│   └── index.js.map   # Source map
├── compiler/
│   └── contract-info.json
├── zkir/
│   └── *.zkir
└── keys/
    ├── *.prover
    └── *.verifier
```

### The Contract API

```typescript
import { myContract } from './contract';

const contract = new myContract(witnesses);

// Call a circuit
const tx = await contract.callTx.mint(metadataHash);

// Read state
const totalSupply = contract.state.totalSupply;
```

---

## Implementing Witnesses

Witnesses are the bridge between your Compact contract and private user data.

### In Compact (declaration)

```compact
witness callerAddress(): Bytes<32>;
witness secretKey(): Bytes<32>;
```

### In TypeScript (implementation)

```typescript
import { findDeployedContract } from '@midnight-ntwrk/midnight-js-contracts';
import { createWallet, createProviders } from './utils';

const SEED = process.env.WALLET_SEED;

async function main() {
  const wallet = await createWallet(SEED);
  const providers = await createProviders(wallet);

  const contract = await findDeployedContract(providers, {
    contractAddress: 'addr...',
    compiledContract: await getCompiled('contract'),
  });

  // Implement witnesses
  const witnesses = {
    callerAddress: () => {
      // Get the caller's derived address
      return derivedAddress(wallet);
    },
    secretKey: () => {
      // Get secret key for signing (never goes on-chain)
      return wallet.secretKey;
    },
  };

  // Call circuit with witnesses
  const tx = await contract.circuits.transfer(
    { context: providers },
    tokenId,
    newOwner,
    tokenMetaHash
  );
}
```

### Witness Context

```typescript
interface WitnessContext {
  transactionId: string;
  proposer: Uint8Array;
  nonce: bigint;
}
```

---

## Circuit Invocation

### Impure Circuits

These access ledger state:

```typescript
// Get context from providers
const context = await providers.midnight();

const result = await contract.circuits.mint(context, metadataHash);

console.log('Transaction:', result.transactionId);
```

### Pure Circuits

These don't access ledger state:

```typescript
// Pure circuits don't need context
const hash = contract.circuits.hashData(data);

console.log('Hash:', hash);
```

### Return Values

```typescript
const result = await contract.circuits.getValue(context);

console.log(result.value);  // The returned value
```

---

## Reading Ledger State

### Simple Fields

```typescript
const state = contract.state;

console.log('Total supply:', state.totalSupply);
console.log('Owner:', state.owner);
```

### Maps

```typescript
// Check membership
const hasKey = state.tokenCommitments.member(tokenId);

// Get value
const commitment = state.tokenCommitments.lookup(tokenId);

// Get root (MerkleTree)
const root = state.merkleTree.root();
```

### Counters

```typescript
const count = state.counter.value();
```

---

## SDK Usage

### Installation

```bash
npm install @midnight-ntwrk/midnight-js-contracts @midnight-ntwrk/midnight-js-sdk
```

### Basic Setup

```typescript
import {
  createWallet,
  createProviders,
  findDeployedContract,
} from '@midnight-ntwrk/midnight-js-contracts';

async function setup() {
  // Create wallet from seed
  const wallet = createWallet('your-seed-phrase');

  // Create providers (connects to chain)
  const providers = await createProviders(wallet);

  return { wallet, providers };
}
```

### Deployment

```typescript
import { deployContract } from '@midnight-ntwrk/midnight-js-contracts';

async function deploy(providers, initialState) {
  const contract = await deployContract(providers, {
    initialState,
    compiledContract: await getCompiled('contract'),
  });

  console.log('Deployed at:', contract.address);
  return contract;
}
```

---

## Error Handling

```typescript
try {
  const tx = await contract.circuits.mint(context, metadataHash);
} catch (error) {
  if (error.message.includes('Assertion failed')) {
    console.log('Circuit assertion failed');
  } else if (error.message.includes('Insufficient balance')) {
    console.log('Not enough tokens');
  } else {
    throw error;
  }
}
```

---

## Type Definitions

The `.d.ts` file tells you what's available:

```typescript
export interface CircuitResults<PS, Returns> {
  state: PS;           // Post-state
  returns: Returns;   // Return values
}

export interface Witnesses {
  callerAddress(context: WitnessContext): [PS, Uint8Array];
  secretKey(context: WitnessContext): [PS, Uint8Array];
}

export interface Ledger {
  readonly totalSupply: bigint;
  readonly owner: Uint8Array;
  tokenCommitments: {
    member(key: bigint): boolean;
    lookup(key: bigint): Uint8Array;
  };
}
```

---

## Quick Recap

- Compile produces TypeScript bindings in `contract/`
- Witnesses are implemented in TypeScript, declared in Compact
- Call circuits via `contract.circuits.method(context, args)`
- Read state via `contract.state.field`
- Use SDK for wallet and deployment

---

## Cross-Links

- **Previous:** [Writing A Contract](./chapter-14.md)  Contract structure
- **Next:** [Constructors](./chapter-16.md)  Contract initialization
- **See also:** [Testing and Debugging](./chapter-17.md)  Troubleshooting