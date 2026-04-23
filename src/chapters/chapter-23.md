# Operators Reference

This note is a quick reference for all operators in Compact.

> **Docs:** [Compact Reference](https://docs.midnight.network/compact/reference/compact-reference)

---

## Arithmetic Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `+` | Addition | `a + b` |
| `-` | Subtraction | `a - b` |
| `*` | Multiplication | `a * b` |
| `/` | Division | `a / b` |
| `%` | Modulo | `a % b` |

### Field Arithmetic

Field operations are modulo the prime field order:

```compact
const x: Field = 10;
const y: Field = 3;
const z = x / y;  // 10 * inverse(3) mod p
const r = x % y;  // Remainder
```

### Uint Arithmetic

Uint operations wrap at the type boundary:

```compact
const x: Uint<8> = 255;
const y = x + 1;  // 0 (wrapped)
```

---

## Comparison Operators

| Operator | Meaning | Types |
|----------|---------|-------|
| `==` | Equal | All |
| `!=` | Not equal | All |
| `<` | Less than | Uint&lt;n&gt;, Uint&lt;0..n&gt; |
| `<=` | Less or equal | Uint&lt;n&gt;, Uint&lt;0..n&gt; |
| `>` | Greater than | Uint&lt;n&gt;, Uint&lt;0..n&gt; |
| `>=` | Greater or equal | Uint&lt;n&gt;, Uint&lt;0..n&gt; |

> **Note:** Field only supports `==` and `!=`. Use Uint types for comparisons.

---

## Boolean Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `&&` | Logical AND | `a && b` |
| `||` | Logical OR | `a \|\| b` |
| `!` | Logical NOT | `!a` |

Short-circuit evaluation:

```compact
const valid = x != 0 && y / x > threshold;
```

---

## Bitwise Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `&` | Bitwise AND | `a & b` |
| `\|` | Bitwise OR | `a \| b` |
| `^` | Bitwise XOR | `a ^ b` |
| `~` | Bitwise NOT | `~a` |

---

## Assignment Operators

| Operator | Example | Meaning |
|----------|---------|---------|
| `=` | `x = 5` | Simple assignment |
| `+=` | `x += 1` | x = x + 1 |
| `-=` | `x -= 1` | x = x - 1 |
| `*=` | `x *= 2` | x = x * 2 |

---

## Type Cast

| Operator | Meaning |
|----------|---------|
| `as T` | Cast to type T |

```compact
const x: Uint<64> = 42;
const y = x as Field;                    // Widen
const z = x as Uint<0..100>;             // Narrow
```

---

## Other Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `? :` | Ternary | `a > b ? a : b` |
| `->` | Function type | `(x: T) -> U` |
| `[...]` | Array literal | `[1, 2, 3]` |
| `...` | Spread | `...vec` |

---

## Precedence

Highest to lowest:

1. `()` `[]` `.`
2. `!` `~` `as`
3. `*` `/` `%`
4. `+` `-`
5. `<<` `>>`
6. `<` `<=` `>` `>=`
7. `==` `!=`
8. `&`
9. `^`
10. `|`
11. `&&`
12. `||`
13. `? :`
14. `=` `+=` etc

---

## Quick Reference

```compact
// Arithmetic
a + b - c * d / e % f

// Comparison
x == y != z > w >= 0 <= 255

// Boolean
flag && !disabled || hidden

// Ternary
max = a > b ? a : b
```

---

## Cross-Links

- **Previous:** [Keywords Reference](./chapter-20.md)  Keyword meanings
- **See also:** [Data Types](./chapter-07.md)  Type system