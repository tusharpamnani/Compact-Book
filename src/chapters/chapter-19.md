# Compact Grammar

This note is a readable reference for Compact's syntax, the rules that shape what you write. The formal grammar is EBNF; this note translates it into what you actually type.

> **Goal after this note:** Read Compact fluently. Know what's legal syntax vs. what's a language rule.
> **Docs:** [Compact Grammar](https://docs.midnight.network/compact/reference/compact-grammar)
> **Examples:** [12.01 Patterns](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/12.01.patterns.compact) · [12.02 Expressions](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/12.02.expressions.compact) · [12.03 Arrow Functions](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/12.03.arrows.compact)

---

## Intuition First

Compact's grammar is deliberately TypeScript-adjacent, but the differences are load-bearing. TypeScript lets you write almost anything, Compact enforces structure at every level:

- Every expression has a type known at compile time.
- Every program element is one of a fixed set of forms.
- Operators have a fixed precedence order.
- Some TypeScript patterns (`return` in loops, mixed field separators, two-armed `if` as the "then" of another `if`) are simply not valid.

The grammar isn't a style guide. It's the contract between you and the compiler. If your code follows the grammar, it compiles. If it doesn't, it doesn't.

---

## Program Structure

A Compact program is a flat sequence of program elements. There's no nesting at the top level, modules contain elements, but the program itself is just one element after another:

```
program → program-element ⋯ program-element
```

Valid elements (in any order, subject to scope rules):

| Element | Keyword(s) | Purpose |
|---------|-----------|---------|
| Pragma | `pragma` | Version constraint |
| Module | `module` | Namespace block |
| Import | `import` | Bring in another module |
| Include | `include` | Inline splice from file |
| Struct | `struct` | Record type |
| Enum | `enum` | Sum type |
| Type alias | `type` / `new type` | Alias |
| Ledger | `ledger` | On-chain state |
| Witness | `witness` | Private input |
| Constructor | `constructor` | Init |
| Export | `export` | Entry point marker |

Module order rule: a module must be **defined before it is imported**. Circular imports are not allowed.

---

## Pragma

```
pragma language_version >= 0.22;
pragma compiler_version >= 0.30.0 && !0.30.1;
```

Version expressions support `||`, `&&`, `!`, `<`, `<=`, `>=`, `>`, and grouping with `()`. Both `major.minor` and `major.minor.patch` forms are valid.

Always put the pragma first. It's the first thing the compiler reads.

---

## Module

```
module Math<T> {
  export circuit add(a: T, b: T): T { return a + b; }
  circuit helper(x: T): T { return x; }  // private to Math
}
```

Generic modules are specialized at import time:

```compact
import Math<Field, 4>;
```

`export` makes a binding visible outside the module. Without it, the binding is private.

---

## Import

```
import Math;                          // all exports
import { add } from Math;             // specific
import { add as plus } from Math;    // renamed
import Math prefix M$;                // prefixed: M$add
import Math<Field, 4>;               // specialized
import "utils/Math";                 // from file path
```

File imports look for `.compact` in the same directory or relative path. Set the search path with `--compact-path` or `COMPACT_PATH`.

---

## Ledger Declaration

```
ledger count: Counter;
export ledger owner: Bytes<32>;
export sealed ledger config: Uint<32>;
```

| Form | Meaning |
|------|--------|
| `ledger x: T` | Basic field, non-exported |
| `export ledger x: T` | Readable from TypeScript |
| `sealed ledger x: T` | Writeable only in constructor |
| `export sealed ledger x: T` | Both |

---

## Witness Declaration

```
witness secretKey(): Bytes<32>;
witness getItem<T>(index: Uint<32>): T;
```

Witnesses have no body in Compact. The body is provided by the TypeScript DApp. Generics are supported.

---

## Constructor

```
constructor(sk: Bytes<32>, v: Uint<64>) {
  authority = disclose(publicKey(round, sk));
  value = disclose(v);
}
```

Runs once on deployment. Parameters come from the deploy transaction. Use `disclose()` for values that should be public from the start.

---

## Circuit Definition

```
circuit add(a: Field, b: Field): Field { return a + b; }
export circuit get(): Uint<64> { return value; }
export pure circuit hash<T>(v: T): Bytes<32> {
  return persistentHash<T>(v);
}
```

| Modifier | Meaning |
|----------|--------|
| `export` | Callable from TypeScript |
| `pure` | No ledger reads/writes, no witness calls |

Generic circuits must be specialized before export.

---

## Types

```
Boolean
Field
Uint<8>
Uint<0..256>
Bytes<32>
Opaque<"string">
Vector<4, Field>
[Field, Boolean, Uint<16>]
Maybe<Field>
Map<Bytes<32>, Uint<64>>
```

| Form | What it is |
|------|-----------|
| `Uint<n>` | Fixed-size unsigned, 0 to 2^n - 1 |
| `Uint<0..n>` | Bounded unsigned, 0 to n-1 |
| `Vector<N, T>` | Fixed-length homogeneous sequence |
| `[T1, T2, ...]` | Fixed-length heterogeneous tuple |
| `tref` | User-defined or stdlib type |

All types are fixed-size at compile time. No `any`, no `unknown`.

---

## Struct and Enum

```
struct Point { x: Field, y: Field }
struct Pair<T> { first: T; second: T }      // semicolons ok
enum State { UNSET, SET }
```

Field separators must be consistent, all commas or all semicolons, not mixed. Trailing separator is allowed.

---

## Type Alias

```
type Hash = Bytes<32>;              // structural, interchangeable
new type UserId = Bytes<32>;       // nominal, requires explicit cast
type V3<T> = Vector<3, T>;       // generic
```

Structural aliases are fully interchangeable with the underlying type. Nominal aliases are distinct types.

---

## Blocks and Statements

```
block → { stmt ⋯ stmt }
stmt  → if ( expr ) stmt
      | stmt0
stmt0 → expr;
      | const cbinding ,⋯, cbinding;
      | if ( expr ) stmt0 else stmt
      | for (const id of nat .. nat) stmt
      | for (const id of expr) stmt
      | return expr;
      | return;
      | block
```

**Critical parsing rule:** `stmt` and `stmt0` are split because a one-armed `if` **cannot** be the "then" branch of a two-armed `if`.

```compact
// VALID
if (a) { x; } else { y; }
if (b) { z; }

// INVALID, syntax error
if (a) if (b) { x; } else { y; }
```

The parser sees `if (a) if (b) { x; }` as the "then" of the outer `if`, and `else { y; }` has no matching `if`. Fix by adding braces:

```compact
if (a) { if (b) { x; } } else { y; }
```

---

## Patterns

```
x                          // simple
[a, b]                     // tuple destructure
[a, , c]                   // skip element
{x, y}                    // struct destructure
{x: myX, y}                // rename
```

Patterns are used in parameter positions and `const` bindings. They let you unpack tuples and structs concisely.

---

## Expressions: Precedence

Operators at higher levels bind more tightly:

| Level | Operators | Notes |
|-------|-----------|-------|
| `expr` | `? :`, `=`, `+=`, `-=` | ternary, assignment |
| `expr0` | `\|\|` | logical or |
| `expr1` | `&&` | logical and |
| `expr2` | `==`, `!=` | equality |
| `expr3` | `<`, `<=`, `>=`, `>` | relational, non-associative |
| `expr4` | `as` | type cast |
| `expr5` | `+`, `-` | additive |
| `expr6` | `*` | multiplicative |
| `expr7` | `!` | logical not (prefix) |
| `expr8` | `[i]`, `.field`, `.method()` | indexing, field access |
| `expr9` | function calls, `map`, `fold`, literals | highest |

**Non-associative relational operators:** `a < b < c` is a syntax error. Write `(a < b) && (b < c)`.

---

## Expression Forms

```
map(fn, vec)                      // transform
fold(fn, init, vec)               // accumulate
slice<4>(v, start)             // sub-vector
[x, ...y]                       // spread
assert(cond, "msg")             // runtime guard
disclose(expr)                   // explicit disclosure
pad(32, "prefix")              // padded string literal
default<T>                      // default value
```

---

## Anonymous Circuits (Arrow Functions)

```
map((x) => x + 1, v)
map((x: Field): Field => x + 1, v)
map((x) => { return x + 1; }, v)
map(double, v)                   // named circuit reference
```

Three forms:

1. **Expression body:** `=> expr`, compact, good for single expressions
2. **Block body:** `=> block`, for multi-statement logic
3. **Named reference:** `circuitName`, pass a named circuit

Type annotations on arrow parameters are optional.

---

## `const` Binding

```
const x = 42;
const x: Field = 42;
const [a, b] = pair;
const {x, y}: Point = p;
const a = 1, b = 2;           // multiple in one statement
```

Variables are immutable after binding. No `let`, no reassignment.

---

## What the Grammar Doesn't Tell You

The grammar tells you what's syntactically valid. It doesn't tell you what passes the type checker or the witness protection program. Three layers of validation:

1. **Syntax**, Does it parse? (grammar)
2. **Types**, Does it type-check? (type system)
3. **Privacy**, Is disclosure declared? (witness protection)

A program that passes the grammar might still fail at step 2 or 3. The error messages distinguish these.

---

## Common Mistakes

1. **Chained relational operators.** `a < b < c` is a syntax error. Relational operators are non-associative, use explicit parentheses.

2. **`if` without braces on one side of `else`.** `if (a) if (b) { } else { }` parses as an orphan `else`. Use braces.

3. **Struct separator inconsistency.** `struct S { a: T, b: T; }` mixes separators, not allowed. All commas or all semicolons.

4. **Generic without specialization.** `export circuit id<T>(x: T): T { return x; }` is exported but generic, invalid. Specialize first: `circuit idField = id<Field>;`.

5. **Assignment vs equality.** `if (x = 42)` is assignment, not comparison. In Compact this evaluates to `42` (truthy), which is almost certainly not what you want. Use `==`.

---

## Comparison Layer

| Feature | TypeScript | Rust | Compact |
|---------|-----------|------|--------|
| Block body in arrow | `x => { return x; }` | `|x| x` | `=> { }` same |
| Type cast | `(x as T)` | `x as T` | `x as T` same |
| Tuple destructuring | `const [a, b] = x` | `let [a, b] = x` | `const [a, b] = x` same |
| Struct fields | `,` or `;` (flexible) | `,` only | must be consistent |
| Assignment in condition | allowed | disallowed | `x = y` parses but warns |
| Rel chaining | `a < b < c` | `a < b && b < c` | syntax error |
| Generic params | `<T>` | `<T>` | `<T>` or `<#N>` |

---

## Quick Recap

- A program is a flat sequence of elements. A module must be defined before imported.
- Pragma first, everything else in any order.
- Struct/enum fields: all commas or all semicolons, not mixed.
- Relational operators are non-associative: `a < b < c` is a syntax error.
- One-armed `if` cannot be the "then" of a two-armed `if`, use braces.
- Generics must be specialized before export.
- Three validation layers: syntax → types → privacy (disclosure).

---

## Cross-Links

- **Previous:** [Data Types](./chapter-08.md)  Type system
- **Next:** [Keywords Reference](./chapter-13.md)  Every keyword
- **See also:** [Standard Library](./chapter-10.md)  Built-in functions
- **Examples:** [12.01 Patterns](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/12.01.patterns.compact) · [12.02 Expressions](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/12.02.expressions.compact) · [12.03 Arrow Functions](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/12.03.arrows.compact)