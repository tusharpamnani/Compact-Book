# Formatter Usage

This note covers the Compact formatter, a tool that rewrites your code to follow the canonical style guide.

> **Docs:** [Formatter Usage](https://docs.midnight.network/compact/compilation-and-tooling/formatter-usage)

---

## Intuition First

The formatter is part of the Compact toolchain. It takes your source file, applies consistent spacing, indentation, and line breaks, and writes a formatted version.

Formatter is not the same as fixing errors. It makes your code readable and consistent. It doesn't change your logic, only the whitespace.

Two ways to use the formatter:

1. **`compact format`**, via the main CLI (recommended).
2. **`format-compact`**, directly as a standalone binary.

---

## Installation

The formatter comes with the Compact toolchain. If you installed `compact`, you already have `format-compact`:

```bash
compact --version
```

---

## Basic Usage

### Format to Standard Output

```bash
format-compact contracts/contract.compact
```

Prints the formatted code to stdout.

### Format to a File

```bash
format-compact contracts/contract.compact formatted/contract.compact
```

Writes to the specified output file.

### Format In-Place

```bash
format-compact contracts/contract.compact contracts/contract.compact
```

Overwrites the original file with the formatted version.

---

## Command-Line Flags

| Flag | What it does | Default |
|------|------------|---------|
| `--help` | Print help text and exit | - |
| `--version` | Print compiler version | - |
| `--language-version` | Print language version | - |
| `--vscode` | Single-line error messages for VS Code | - |
| `--line-length n` | Target line length | 100 |

### Line Length

```bash
format-compact --line-length 80 contracts/contract.compact
```

Lines longer than `n` are wrapped.

---

## Via the Compact CLI

The main `compact` CLI invokes the formatter:

```bash
compact format contracts/           # format all .compact files in directory
compact format --check contracts/    # check formatting without modifying
```

Use `--check` in CI pipelines to verify formatting:

```bash
compact format --check contracts/
if [ $? -eq 0 ]; then
  echo "Formatting OK"
else
  echo "Formatting issues found"
  exit 1
fi
```

---

## What the Formatter Changes

The formatter applies consistent style:

| Element | Before | After |
|---------|--------|-------|
| Indentation | 2 spaces | 2 spaces (standard) |
| Line breaks | Inconsistent | Consistent |
| Trailing whitespace | Removed | Removed |
| Empty lines | Inconsistent | At most one |
| Array brackets | `[x,y,z]` | `[x, y, z]` |
| Enum variants | `up,down,left,right` | `up, down, left, right` |

The formatter doesn't change:

- Variable names
- Type annotations
- Logic or expressions

---

## Error Handling

If the source file has static errors, the formatter fails:

```bash
$ format-compact contracts/broken.compact
Exception: /path/contracts/broken.compact line 12 char 5:
  type error: expected Uint<64>, got Field
```

The formatter validates syntax before formatting. Fix errors first.

---

## CI Integration

Add formatting checks to your CI pipeline:

```yaml
# .github/workflows/format.yml
name: Format Check
on: [push, pull_request]
jobs:
  format:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install compact
        run: curl ... | sh
      - name: Check formatting
        run: compact format --check contracts/
```

---

## Quick Recap

- `format-compact` rewrites whitespace without changing logic.
- Use `compact format` for convenient batch formatting.
- Use `--check` in CI to verify formatting.
- Fix syntax errors before formatting.
- The formatter ensures consistent style across your codebase.

---

## Cross-Links

- **Next:** [Fixup Usage](./chapter-18.md)  Automatic code migration
- **See also:** [Setting Up the Compiler](./chapter-02.md)  Toolchain installation