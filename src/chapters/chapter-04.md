# Neovim Setup

This note covers setting up Compact language support in Neovim using [compact.vim](https://github.com/1NickPappas/compact.vim), a community-driven plugin.

> **Docs:** [Neovim Setup](https://docs.midnight.network/compact/compilation-and-tooling/neovim-setup)

---

## Intuition First

The compact.vim plugin provides language features for Compact source files in Neovim. It's not an official Midnight plugin, but it's the recommended way to edit `.compact` files with proper syntax highlighting and indentation.

The plugin matches the language features found in the VS Code extension, but for Neovim.

---

## Features

| Feature | What it does |
|---------|-------------|
| Syntax highlighting | Keywords, types, circuits colored |
| Smart indentation | Proper indent for blocks |
| Code folding | Fold circuit definitions |
| Text objects | Select circuit bodies |
| Import navigation | `gf` jumps to imports |
| Compiler integration | `:make` runs the compiler |

---

## Installation

### Using lazy.nvim

Add to your plugins in `init.lua`:

```lua
return require("lazy").setup({
  { "1NickPappas/compact.vim" },
})
```

### Using packer

```lua
use("1NickPappas/compact.vim")
```

### Using vim-plug

```vim
Plug '1NickPappas/compact.vim'
```

---

## Configuration

### File Detection

The plugin automatically detects `.compact` files. No configuration needed.

### Syntax Highlighting

Syntax highlighting works automatically.

### Compiler Integration

Run the compiler from Neovim:

```vim
:compiler compactc
:make %:p
```

---

## Usage

### Opening a Compact File

```bash
nvim contracts/contract.compact
```

The plugin activates automatically based on the `.compact` extension.

### Folding

Fold circuits and blocks:

```vim
za         " toggle fold
zR         " open all folds
zM         " close all folds
```

### Navigation

Jump to imports with `gf`:

```vim
gf         " go to file under cursor
```

---

## Tree-Sitter Integration

The plugin supports tree-sitter for better parsing:

```lua
require("nvim-treesitter.configs").setup({
  ensure_installed = { "compact" },
})
```

The tree-sitter parser provides accurate syntax highlighting and better performance.

---

## Quick Recap

- Install via your plugin manager.
- Syntax highlighting and indentation work automatically.
- Use `:make` to compile from Neovim.
- Review troubleshooting for issues.

---

## Cross-Links

- **Previous:** [Fixup Usage](./chapter-18.md)  Automatic code migration
- **Next:** [VS Code Extension](./chapter-20.md)  VS Code setup
- **See also:** [Setting Up the Compiler](./chapter-02.md)  Toolchain installation