# Control Flow

This note covers conditionals, loops, and assertions in Compact.

> **Docs:** [Compact Reference](https://docs.midnight.network/compact/reference/compact-reference)

---

## Intuition First

Control flow in Compact is constrained by design. No recursion, bounded loops only.

---

## Conditionals

### if/else

```compact
if (condition) {
  // then branch
} else {
  // else branch
}
```

### One-arm if

```compact
if (value > 0) {
  result = true;
}
```

### No else-if

```compact
// wrong - not allowed
if (x == 1) { } else if (x == 2) { }

// correct
if (x == 1) { } else { if (x == 2) { } }
```

---

## Assertions

```compact
assert(condition, "error message");
assert(value > 0, "Must be positive");
```

---

## Loops

### For Loop

```compact
for (const item of collection) {
  // process item
}
```

Bound must be compile-time known.

### fold

```compact
const sum = v.fold((acc, x) => acc + x, 0);
```

### No while

While loops are not allowed.

---

## Return in Loops

Not allowed. Use fold instead:

```compact
// wrong
for (const x of v) { if (x == target) return true; }

// correct
return v.fold((found, x) => found || x == target, false, v);
```

---

## Common Patterns

### Finding an element
```compact
const found = v.fold((acc, x) => acc || x == target, false, v);
```

### Summing
```compact
const total = v.fold((sum, x) => sum + x, 0n);
```

### Maximum
```compact
const max = v.fold((m, x) => x > m ? x : m, default<Field>());
```

---

## Quick Recap

- if/else allowed, no else-if
- assert is your runtime check
- for loops must have compile-time bounds
- Use fold for early returns

---

## Cross-Links

- **Previous:** [Data Types](./chapter-08.md)
- **Next:** [Circuits](./chapter-07.md)