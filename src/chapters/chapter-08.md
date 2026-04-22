# Data Types

This note covers Compact's type system, primitives, composites, and program-defined types.

> **Docs:** [Compact Types](https://docs.midnight.network/compact/reference/compact-reference#compact-types)
> **Examples:** [08.01 Primitives](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/08.01.primitives.compact) · [08.02 Composites](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/08.02.composites.compact)

---

## Intuition First

Compact is statically and strongly typed. Every expression has a type known at compile time. All types have fixed sizes at compile time. No `any`, no implicit `undefined`, no guessing.

The type system serves two purposes in Compact:

1. **Normal type checking**, catching mismatches before runtime.
2. **Privacy enforcement**, the compiler tracks which types contain witness data, and where that data can flow.

Strong typing is what makes privacy enforcement possible. If types were loose, the compiler couldn't track data flow.

---

## Primitive Types

### Boolean

```compact
const flag: Boolean = true;
const other = false;
```

Two values: `true` and `false`. No truthy/falsy conversion, must be explicit.

### Field

The set of unsigned integers up to the order of the native prime field. Values in a Field can only be compared with `==` and `!=`, not with `<`, `<=`, `>`, `>=`.

```compact
const f: Field = 42;
const g = 0xdeadbeef as Field;  // large literals must be cast
```

**Why no comparison?** Field arithmetic is modulo a prime. `<` comparisons in that space don't behave like integer comparisons. Use bounded types (`Uint<0..n>`) if you need comparisons.

### Uint\<n\>: Sized Integer

A fixed-width unsigned integer. Exactly `n` bits.

```compact
const x: Uint<8> = 255;     // 0 to 255
const y: Uint<64> = 1000000;
```

Overflow wraps. `Uint<8>(255) + Uint<8>(1) == 0`.

### Uint<0..n>: Bounded Integer

An unsigned integer with an explicit range. Values outside the range are rejected at compile time.

```compact
const age: Uint<0..150> = 25;   // 0 to 149
const idx: Uint<0..256> = 100;  // 0 to 255
```

`Uint<0..n>` is a subtype of `Uint<0..m>` if `n ≤ m`.

### Bytes\<n\>

Exactly `n` bytes. Used for hashing, keys, identifiers.

```compact
const key: Bytes<32> = pad(32, "midnight:example:key");
const hash: Bytes<32> = persistentHash<Field>(42);
```

Fixed length. Padding fills with zeros if the input is short.

### Opaque

Allows foreign JavaScript data to pass through without inspection by Compact code. Circuits see only a hash.

```compact
witness getMessage(): Opaque<"string">;
export ledger message: Opaque<"string">;

export circuit post(): [] {
  message = disclose(getMessage());
}
```

Circuits cannot inspect the contents, they can only store and retrieve the value. The value is opaque to Compact but transparent in TypeScript.

**Important:** Opaque values are not hidden on-chain. They're plaintext, just not directly readable by circuits.

---

## Composite Types

### Tuples [T1, T2, ..., Tn]

Fixed-length, heterogeneous, positional.

```compact
const pair: [Field, Boolean] = [42, true];
const first = pair[0];   // Field
const second = pair[1];  // Boolean
```

Access by index. Types must match exactly.

### Vector<n, T>

Homogeneous fixed-length sequence.

```compact
const v: Vector<4, Uint<8>> = [1, 2, 3, 4];
const w = [10, 20, 30];  // inferred as Vector<3, ...>
```

Use `map` and `fold` for transformations.

---

## Program-Defined Types

### struct

Named collection of fields. Nominal typing, two structs with the same shape but different names are different types.

```compact
struct Point {
  x: Uint<32>,
  y: Uint<32>,
}

const p = Point { x: 10, y: 20 };
const xVal = p.x;
```

Structs cannot be recursive.

### enum

Named set of variants. The first variant is the default value.

```compact
enum Direction { up, down, left, right }
enum State { UNSET, SET }

const d = Direction.up;
const s = State.SET;
```

Useful for state machines and finite domains.

---

## Type Aliases

### Structural alias

Interchangeable with the underlying type.

```compact
type Pair<T> = [T, T];
type Hash = Bytes<32>;
```

Can use `Hash` anywhere you use `Bytes<32>`, and vice versa.

### Nominal alias

Distinct type requiring explicit cast.

```compact
new type UserId = Bytes<32>;
new type TokenAmount = Uint<64>;
```

Cannot use `Bytes<32>` where `UserId` is expected without a cast. This prevents mixing up IDs and amounts.

---

## Subtyping Rules

| Rule | Meaning |
|------|---------|
| Any `T` is a subtype of itself | Identity |
| `Uint<0..n>` is a subtype of `Uint<0..m>` if `n ≤ m` | Range widening |
| `Uint<0..n>` is a subtype of `Field` if `n-1` is within field range | Field compatibility |
| `[T1, ..., Tn]` is a subtype of `[S1, ..., Sn]` if each `Ti` is a subtype of `Si` | Tuple matching |

Subtyping is used in assignment and parameter passing.

---

## Type Casting

```compact
const x: Uint<64> = 42;
const y = x as Field;          // widen to Field
const z = x as Uint<0..1000>;  // narrow, dynamic error if out of range
const b = someBytes as UserId;   // nominal alias cast
```

- `as T` widens or narrows.
- Narrowing that fails at runtime produces a transaction error.
- Nominal aliases require explicit cast.

---

## Default Values

Every type has a compile-time-known default:

| Type | Default |
|------|---------|
| `Boolean` | `false` |
| `Uint<n>`, `Uint<0..n>`, `Field` | `0` |
| `Bytes<n>` | `n` zero bytes |
| `[T1, ..., Tn]` | tuple of defaults |
| `Vector<n, T>` | vector of defaults |
| `struct` | each field to default |
| `enum` | first variant |
| `Opaque<"string">` | zero-length string |
| `Opaque<"Uint8Array">` | zero-length array |

```compact
const empty = default<Bytes<32>>;
const zero = default<Uint<64>>;
```

Used when initializing ledger fields without a constructor.

---

## Common Mistakes

1. **Using `Field` when you need comparisons.** Field values can only be compared with `==` and `!=`. Use `Uint<0..n>` for ordered comparisons.

2. **Assuming unbounded `Uint`.** `Uint<64>` is exactly 64 bits, wrapping at overflow. Use bounded types if you need range checking.

3. **Confusing structural and nominal aliases.** `type Hash = Bytes<32>` is structural, fully interchangeable. `new type UserId = Bytes<32>` is nominal, requires explicit cast.

4. **Forgetting that `as Uint<0..n>` can fail at runtime.** Narrowing to a bounded type with a value outside the range produces a transaction error, not a compiler error.

5. **Using `Bytes<n>` when you need padding.** `Bytes<32>` is exactly 32 bytes. Use `pad(32, str)` to create fixed-length byte vectors from strings.

---

## Comparison Layer

| Concept | TypeScript | Rust | Compact |
|---------|-----------|------|--------|
| Sized int | N/A | `u64`, `u8` | `Uint<64>`, `Uint<8>` |
| Bounded int | `number` (runtime check) | N/A | `Uint<0..n>` |
| Byte array | `Buffer`, `Uint8Array` | `[u8; 32]` | `Bytes<32>` |
| Tuple | `[type1, type2]` | `(T1, T2)` | `[T1, T2]` |
| Struct | `class`, `interface` | `struct` | `struct` |
| Enum | `enum` | `enum` | `enum` |
| Type alias | `type A = B` | `type A = B` | `type A = B` or `new type A = B` |
| Dynamic type | `any` | N/A | **not available** |

---

## Quick Recap

- All types are fixed-size at compile time. No `any`.
- `Field` supports only `==` and `!=`. Use `Uint<0..n>` for ordered comparisons.
- `Uint<0..n>` narrows to bounded range. `as Uint<0..n>` can fail at runtime.
- `new type` creates nominal aliases, requires explicit cast.
- `type` creates structural aliases, fully interchangeable.
- Every type has a default value. Use `default<T>()`.
- Opaque values are not hidden, they're just not directly readable by circuits.

---

## Cross-Links

- **Previous:** [Explicit Disclosure](./chapter-07.md)  Disclosure boundary
- **Next:** [Ledger ADTs](./chapter-09.md)  Collection types for ledger state
- **See also:** [Standard Library](./chapter-10.md)  Built-in types
- **Examples:** [08.01 Primitives](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/08.01.primitives.compact) · [08.02 Composites](https://github.com/tusharpamnani/Compact-Book/blob/main/examples/08.02.composites.compact)