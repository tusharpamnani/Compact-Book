# Modules and Imports

This note covers organizing code across files using Compact's static module system.

> **Docs:** [Modules, Exports, and Imports](https://docs.midnight.network/compact/reference/compact-reference#modules-exports-and-imports)
> **Examples:** [11.01 Module Definition](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/11.01.module.compact) · [11.02 Imports](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/11.02.imports.compact) · [11.03 Exports](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/11.03.exports.compact)

---

## Intuition First

Compact uses a static module system, modules are defined before use, and file resolution is determined at compile time. There are no runtime imports or dynamic module loading.

This is simpler than it sounds. If you're coming from TypeScript or Rust, you already know the pattern:

1. Define a module.
2. Export what you want to expose.
3. Import it where you need it.
4. The compiler resolves files at compile time.

---

## Defining a Module

```compact
module Math {
  export circuit add(a: Field, b: Field): Field {
    return a + b;
  }

  export circuit mul(a: Field, b: Field): Field {
    return a * b;
  }

  circuit helper(x: Field): Field {  // not exported
    return x + 1;
  }
}
```

Bindings inside a module are invisible outside unless explicitly exported. `helper` is private to `Math`.

---

## Exporting from a Module

Two ways to export:

```compact
// Inline export
module M {
  export struct Point { x: Field, y: Field }
  export circuit distance(p: Point): Field { ... }
}

// Separate export
module M {
  struct Point { x: Field, y: Field }
  circuit distance(p: Point): Field { ... }
  export { Point, distance };
}
```

Both are equivalent. Inline export is more compact for small modules. Separate export is useful for controlling the public API.

---

## Importing

```compact
import Math;                    // everything exported by Math
import { add } from Math;       // specific binding
import Math prefix Math$;        // with prefix (Math$add)
import { add as plus } from Math; // renamed
```

Prefix notation (`Math$add`) is useful when you have name conflicts across modules.

---

## Generic Modules

```compact
module Container<T, #N> {
  export circuit first(v: Vector<N, T>): T { return v[0]; }
  export circuit last(v: Vector<N, T>): T { return v[N - 1]; }
}

import Container<Field, 4>;
```

Generic modules must be specialized at import time. `Container<Field, 4>` is a concrete module.

---

## Importing from Files

```compact
import MyModule;
// looks for MyModule.compact in the same directory

import "utils/Math";
// looks for utils/Math.compact relative to the importing file
```

Rules:
- File must contain exactly one module definition.
- If not found, it's a compile error.
- Search path is set via `--compact-path` or `COMPACT_PATH`.

---

## Include Files

`include` splices contents directly into the current file:

```compact
include "shared/types.compact";
```

Unlike `import`, `include` is text substitution, the included contents become part of the file. Use for sharing type definitions, enums, and constants.

---

## Top-Level Exports

```compact
export circuit transfer(to: Bytes<32>, amount: Uint<64>): [] { ... }
export struct TokenInfo { name: Bytes<32>, supply: Uint<64> }
export ledger totalSupply: Uint<64>;
```

Top-level exports are the contract's public API. They're callable from TypeScript.

Rules:
- No two exported circuits can share the same name.
- Generic circuits cannot be exported, they must be specialized first.

---

## Module Order

**Important:** A module must be **defined before** any import of that module.

```compact
// INCORRECT
import State;  // error: State not yet defined
module State { ... }

// CORRECT
module State { ... }
import State;
```

This is a compile-time requirement, not a style choice. Circular dependencies are not allowed.

---

## Practical Example

**`state.compact`**

```compact
module State {
  enum STATUS { UNSET, SET }
  ledger status: STATUS;
  ledger value: Field;

  export circuit init(v: Field): [] {
    value = disclose(v);
    status = STATUS.SET;
  }

  export circuit get(): Field {
    assert(status == STATUS.SET, "Not initialized");
    return value;
  }
}
```

**`main.compact`**

```compact
pragma language_version 0.22;
import CompactStandardLibrary;
import State;

constructor(v: Field) {
  init(v);
}

export circuit getValue(): Field {
  return State.get();
}
```

`main.compact` defines no state itself, it just imports and re-exports `State`. This separation of concerns keeps contracts organized.

---

## Common Mistakes

1. **Importing before defining.** The compiler requires module definitions to come before imports. Move the `module` block above the `import` statement.

2. **Assuming `include` works like `import`.** `include` is text substitution. `import` is a reference. Use `include` for types, `import` for modules.

3. **Exporting generic circuits directly.** Generic circuits must be specialized before export. Move specialization to a non-exported circuit.

4. **Forgetting the search path.** If `import "utils/Math"` fails, the compiler can't find the file. Set `COMPACT_PATH` or use `--compact-path`.

5. **Name conflicts across modules.** Use `prefix` notation (`Math$add`) or explicit renaming (`import { add as plus } from Math`) to avoid conflicts.

---

## Comparison Layer

| Concept | TypeScript | Rust | Compact |
|---------|-----------|------|--------|
| Namespace | module, import | mod, use | module, import |
| Visibility | `export` | `pub` | `export` |
| Private | default | default | default |
| Circular deps | runtime error | compile error | compile error |
| Generic modules | `module<T>` | `mod<T>` | `module<T, #N>` |
| Text inclusion | N/A | N/A | `include` |

---

## Quick Reference

| Syntax | Effect |
|--------|--------|
| `module M { ... }` | Define module `M` |
| `export circuit f(...)` | Export from module or top-level |
| `export { f, g }` | Export multiple bindings |
| `import M` | Import all exports of `M` |
| `import { f } from M` | Import specific binding |
| `import M prefix P$` | Import with prefix |
| `import { f as g } from M` | Rename on import |
| `import M<T, 4>` | Generic module specialization |
| `import "path/M"` | Import from file path |
| `include "file.compact"` | Splice file inline |
| `import CompactStandardLibrary` | Built-in stdlib |

---

## Cross-Links

- **Previous:** [Standard Library](./chapter-10.md)  Built-in utilities
- **Next:** [Compact Grammar](./chapter-12.md)  Syntax rules
- **See also:** [Writing a Contract](./chapter-03.md)  Contract structure
- **See also:** [Data Types](./chapter-08.md)  Type definitions
- **Examples:** [11.01 Module](../examples/11.01.module.compact) · [11.02 Imports](../examples/11.02.imports.compact) · [11.03 Exports](../examples/11.03.exports.compact)