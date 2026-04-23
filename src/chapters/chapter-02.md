# Setting Up the Compiler

This note walks through installing the Compact toolchain and getting your first contract compiled.

> **Goal after this note:** Have `compact` installed and able to compile a basic contract.

---

## Intuition First

Compact has two tools:

- **`compact`**, The CLI you use day-to-day (compile, format, fixup).
- **`compactc`**, The actual compiler. You invoke it via `compact compile`.

The toolchain has one non-obvious dependency: Docker. The proof server generates ZK proofs locally and requires Docker Desktop running. This is the step most people skip.

---

## The Three Components

| Component | What it is | Runs where |
|-----------|-----------|-----------|
| `compact` CLI | Your interface to the toolchain | Your terminal |
| `compact compile` | The actual compiler | Invoked by `compact` |
| Proof server (Docker) | Generates ZK proofs locally | Docker Desktop |

You interact with the CLI. The CLI invokes the compiler. The proof server generates proofs when you call circuits.

---

## Prerequisites

Before installing Compact, ensure you have:

- **Docker Desktop**, Required for the proof server
- **Chrome browser**, For the Lace Wallet extension
- **Visual Studio Code**, For the syntax extension
- **Lace Wallet (Chrome extension)**, For wallet integration during development

> **Note:** Linux and macOS are directly supported. On Windows, use WSL.

---

## Step 1: Install the `compact` CLI

```bash
curl --proto '=https' --tlsv1.2 -LsSf https://github.com/midnightntwrk/compact/releases/latest/download/compact-installer.sh | sh
```

Add the binary to your PATH:

```bash
export PATH="$HOME/.compact/bin:$PATH"
```

**Where it installs:** The installer places `compact` at `$HOME/.compact/bin/compact`. Add this directory to your PATH permanently (in `.bashrc` or equivalent).

Verify:

```bash
compact --version
```

---

## Step 2: Install the Compiler

```bash
compact update
```

This downloads and installs the current compiler version.

Verify both:

```bash
compact --version
compact compile --version
```

---

## Step 3: Start the Proof Server

The proof server generates ZK proofs locally. It requires Docker Desktop running.

```bash
docker run -p 6300:6300 midnightntwrk/proof-server:8.0.3 midnight-proof-server -v
```

What this does:

- Starts the proof server container on port 6300
- The server listens at `http://localhost:6300`

Configure Lace Wallet to use it:

1. Open Lace Wallet
2. Go to **Settings → Midnight**
3. Select **Local (http://localhost:6300)**

> **Tip:** Keep Docker Desktop running whenever you're developing. The proof server must be up when you call circuits.

---

## Step 4: Install the VS Code Extension

1. Download the VSIX package:
   ```
   https://raw.githubusercontent.com/midnight-ntwrk/releases/gh-pages/artifacts/vscode-extension/compact-0.2.13/compact-0.2.13.vsix
   ```
2. In VS Code: **Extensions → Install from VSIX** → select the downloaded file

This adds syntax highlighting for `.compact` files.

---

## Compiling a Contract

With everything installed, compile your first contract:

```bash
compact compile contracts/counter.compact contracts/managed/counter
```

Output directory:

```
contracts/managed/counter/
├── contract/             # TypeScript runtime
│   ├── index.js
│   ├── index.d.ts
│   └── index.js.map
├── compiler/             # Metadata
│   └── contract-info.json
├── zkir/                 # ZK circuits
│   ├── *.zkir
│   └── *.bzkir
└── keys/                 # Proving keys
    ├── *.prover
    └── *.verifier
```

**Development shortcut:** Skip ZK key generation during development, it's slow:

```bash
compact compile --skip-zk contracts/counter.compact contracts/managed/counter
```

Re-enable for production builds.

---

## Other Commands

```bash
compact format contracts/           # format all .compact files
compact format --check contracts/    # check formatting (CI)
compact fixup contracts/            # apply source-level fixups
compact list --installed      # list installed versions
```

---

## Troubleshooting

| Error | Fix |
|-------|-----|
| `compact: command not found` | Add `$HOME/.compact/bin` to PATH |
| Docker connection errors | Ensure Docker Desktop is running |
| Port 6300 in use | Use a different port: `-p 6301:6300` |
| `compact update` fails | Check your internet connection; try again |

---

## Standard Library

The standard library is built into the compiler, it's not a file on disk. Import it at the top of every contract:

```compact
import CompactStandardLibrary;
```

This gives you `Maybe`, `Either`, hashing functions, token operations, and more.

---

## Quick Recap

- `compact` is the CLI. `compact compile` invokes the compiler.
- Install via the shell script. Add `$HOME/.compact/bin` to your PATH.
- Docker Desktop must be running for the proof server.
- Start the proof server: `docker run -p 6300:6300 midnightntwrk/proof-server:8.0.3 midnight-proof-server -v`
- Use `--skip-zk` during development to skip slow key generation.
- Import `CompactStandardLibrary` at the top of every contract.

---

## Cross-Links

- **Previous:** [Why Compact](./chapter-01.md), Design goals
- **Next:** [Writing a Contract](./chapter-03.md), Write your first contract
- **See also:** [Official Installation Docs](https://docs.midnight.network/getting-started/installation)