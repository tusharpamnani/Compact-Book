# VS Code Extension

This note covers the Visual Studio Code extension for Compact, which provides syntax highlighting, snippets, and integrated compilation.

> **Docs:** [VS Code Extension](https://docs.midnight.network/compact/compilation-and-tooling/vscode-plugin)

---

## Intuition First

The VS Code extension adds Compact language support to Visual Studio Code. It highlights keywords, provides code snippets, and integrates with the compiler for error reporting.

Installed via a VSIX file, not the VS Code marketplace. The extension is maintained by the Midnight team.

---

## Installation

### Download the VSIX

Download the latest release:

```bash
curl -LO https://raw.githubusercontent.com/midnight-ntwrk/releases/gh-pages/artifacts/vscode-extension/compact-0.2.13/compact-0.2.13.vsix
```

### Install in VS Code

1. Open VS Code
2. Go to **Extensions** (Ctrl+Shift+X)
3. Click **Install from VSIX...**
4. Select the downloaded file

---

## Features

| Feature | What it does |
|---------|-------------|
| Syntax highlighting | Keywords, types, circuits colored |
| Code snippets | Insert common patterns |
| Error highlighting | Inline compiler errors |
| Build integration | Compile from VS Code |
| File templates | New Compact files |

---

## Syntax Highlighting

The extension recognizes:

- **Keywords:** `circuit`, `witness`, `export`, `ledger`, `enum`, `struct`, `module`, `import`, `assert`
- **Types:** `Uint<n>`, `Uint<0..n>`, `Field`, `Boolean`, `Bytes<n>`, `Map`, `Vector`
- **Literals:** strings, numbers, booleans
- **Comments:** `//` and `/* */`

---

## Code Snippets

Type the snippet prefix and press Tab:

| Prefix | Inserts |
|--------|---------|
| `ledger` | State declaration |
| `constructor` | Constructor block |
| `circuit` | Circuit function |
| `witness` | Witness function |
| `stdlib` | Standard library import |
| `if` | If statement |
| `for` | For loop |
| `fold` | Fold expression |
| `enum` | Enum definition |
| `struct` | Struct definition |
| `module` | Module definition |
| `assert` | Assert statement |
| `compact` | Full contract template |

---

## Building from VS Code

### Add a Build Script

In `package.json`:

```json
{
  "scripts": {
    "compact": "compact compile --vscode ./contracts/myContract.compact ./contracts/managed/myContract"
  }
}
```

The `--vscode` flag formats errors for VS Code.

### Compile

```bash
yarn compact
```

Errors appear in the **Problems** panel.

### Task Configuration

For integrated building, create `.vscode/tasks.json`:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Compile Compact",
      "type": "shell",
      "command": "npx compact compile --vscode --skip-zk ${file} ${workspaceFolder}/contracts/managed",
      "group": "build",
      "problemMatcher": [
        "$compactException",
        "$compactInternal",
        "$compactCommandNotFound"
      ]
    }
  ]
}
```

Then press Ctrl+Shift+B to build.

---

## Creating a New Contract

1. Open the command palette (Ctrl+Shift+P)
2. Type **Snippets: Fill File with Snippet**
3. Select **Compact**

This creates a full contract template:

```compact
pragma language_version 0.22;

import CompactStandardLibrary;

export ledger state: State;

enum State { UNSET, SET }

constructor() {
}

export circuit init(): [] {
}

export circuit interact(): [] {
}
```

---

## Quick Recap

- Install via VSIX file.
- Syntax highlighting works automatically.
- Use snippets for common patterns.
- Add `--vscode` to compile command for error integration.
- Use Ctrl+Shift+B to build.

---

## Cross-Links

- **Previous:** [Neovim Setup](./chapter-19.md)  Neovim editor
- **Next:** [Testing and Debugging](./chapter-14.md)  Error handling
- **See also:** [Setting Up the Compiler](./chapter-02.md)  Toolchain installation