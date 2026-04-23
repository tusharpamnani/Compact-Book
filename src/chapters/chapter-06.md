# Fixup Usage

This note covers the Compact fixup tool, which automatically migrates your code when the language changes.

> **Docs:** [Fixup Usage](https://docs.midnight.network/compact/compilation-and-tooling/fixup-usage)

---

## Intuition First

The fixup tool exists because Compact evolves. When the language adds new syntax or changes existing behavior, old code may not compile.

Fixup reads your source file, applies automatic migrations for known language changes, writes the updated code. It's like a linter that fixes things for you.

Fixup doesn't make arbitrary changes. It only applies known transformations from documented language changes.

---

## Installation

Fixup comes with the Compact toolchain:

```bash
fixup-compact --version
```

---

## Basic Usage

### Fixup to Standard Output

```bash
fixup-compact contracts/contract.compact
```

Prints the updated code to stdout.

### Fixup to a File

```bash
fixup-compact contracts/contract.compact fixed/contract.compact
```

Writes to the specified output file.

### Fixup In-Place

```bash
fixup-compact contracts/contract.compact contracts/contract.compact
```

Overwrites the original.

> **Warning:** Always output to a different file first and review the changes. Fixup can introduce errors if the transformation doesn't apply to your code.

---

## Command-Line Flags

| Flag | What it does | Default |
|------|------------|---------|
| `--help` | Print help text and exit | - |
| `--version` | Print compiler version | - |
| `--language-version` | Print language version | - |
| `--vscode` | Single-line error messages | - |
| `--update-Uint-ranges` | Adjust Uint range endpoints | - |
| `--compact-path` | Module search paths | `$COMPACT_PATH` or empty |
| `--trace-search` | Print module search progress | - |
| `--line-length n` | Target line length | 100 |

### Update Uint Ranges

```bash
fixup-compact --update-Uint-ranges contracts/contract.compact
```

This transforms `Uint<0..n>` declarations where `n` is a constant.

### Compact Search Path

```bash
fixup-compact --compact-path /path/to/modules contracts/contract.compact
```

Sets directories where the fixup tool searches for imported modules.

---

## Via the Compact CLI

The main `compact` CLI invokes fixup:

```bash
compact fixup contracts/           # fixup all .compact files in directory
```

---

## Reviewing Changes

Always review before committing:

```bash
# 1. Fixup to a new file
fixup-compact contracts/contract.compact fixed/contract.compact

# 2. Compare
diff contracts/contract.compact fixed/contract.compact

# 3. If OK, replace
mv fixed/contract.compact contracts/contract.compact

# 4. Recompile
compact compile contracts/contract.compact contracts/managed/contract
```

Never run fixup and commit without reviewing.

---

## Error Handling

If the source file has errors that fixup can't handle:

```bash
$ fixup-compact contracts/broken.compact
Exception: /path/contracts/broken.compact line 12 char 5:
  type error: expected Uint<64>, got Field
```

Fixup reports static errors and exits without producing output.

---

## Before Upgrading the Toolchain

```bash
# 1. Update compact
compact update

# 2. Run fixup on your codebase
compact fixup contracts/

# 3. Recompile
compact compile contracts/contract.compact contracts/managed/contract

# 4. Run tests
npm test
```

---

## Quick Recap

- Fixup automatically migrates code for language changes.
- Always review changes before committing.
- Recompile and test after applying fixup.
- Fixup doesn't fix logic errors, only deprecated syntax.

---

## Cross-Links

- **Previous:** [Formatter Usage](./chapter-17.md)  Code formatting
- **Next:** [Neovim Setup](./chapter-18.md)  Editor integration
- **See also:** [Testing and Debugging](./chapter-14.md)  Error handling