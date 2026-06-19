# Rust API Implementation Guideline

This document translates the [language-agnostic API meta-definition](api-guideline.md) into concrete Rust patterns,
conventions, and ready‑to‑copy code templates.
It aims to keep SDK APIs consistent, ergonomic, and maintainable across modules.

## Type Mapping

| Generic Type      | Rust Type                                       | Notes                                       |
|-------------------|-------------------------------------------------|---------------------------------------------|
| `string`          | `String` or `&str`                              | Use `String` for owned, `&str` for borrowed |
| `intX`            | `i8`, `i16`, `i32`, `i64`, `i128`               | -                                           |
| `uintX`           | `u8`, `u16`, `u32`, `u64`, `u128`               | -                                           |
| `double`          | `f64`                                           | -                                           |
| `decimal`         | Third-party crate (e.g., `rust_decimal`)        | -                                           |
| `bool`            | `bool`                                          | -                                           |
| `bytes`           | `Vec<u8>` or `&[u8]`                            | -                                           |
| `list<TYPE>`      | `Vec<TYPE>`                                     | -                                           |
| `set<TYPE>`       | `HashSet<TYPE>` or `BTreeSet<TYPE>`             | -                                           |
| `map<KEY, VALUE>` | `HashMap<KEY, VALUE>` or `BTreeMap<KEY, VALUE>` | -                                           |
| `date`            | `chrono::NaiveDate`                             | Using `chrono` crate                        |
| `time`            | `chrono::NaiveTime`                             | Using `chrono` crate                        |
| `dateTime`        | `chrono::NaiveDateTime`                         | Using `chrono` crate                        |
| `zonedDateTime`   | `chrono::DateTime<Tz>`                          | Using `chrono` crate                        |

## Inheritance and Nullability Narrowing

Rust has no class inheritance; the meta-language's parent / child relationship is mapped to
**traits + concrete structs**. The
[Narrowing inherited nullability](api-guideline.md#narrowing-inherited-nullability) rule —
where a child re-declares an inherited `@@nullable` field as non-`@@nullable` via
`@@override` — maps naturally onto this split:

- The trait method on the parent returns `Option<T>` (matches the parent's `@@nullable`
  contract).
- The concrete child struct stores the value as `T` directly (matches the child's narrowed
  contract).
- The trait `impl` on the child wraps the stored value in `Some(...)` so that calls through
  the trait still satisfy the parent's `Option<T>` signature.
- Callers who hold the concrete type access the field directly (`addr.num` → `u64`); callers
  who hold a trait object go through the trait method (`addr.num()` → `Option<u64>`).

```text
// Meta-language
abstraction Identifier {
    @@immutable @@nullable num: uint64
}

@@finalType
NumericIdentifier extends Identifier {
    @@immutable @@override num: uint64
}
```

```rust
// Parent — trait carrying the @@nullable contract.
pub trait Identifier {
    fn num(&self) -> Option<u64>;
}

// Child — concrete struct stores `u64` directly (narrowed, never absent on this type).
pub struct NumericIdentifier {
    pub num: u64,
}

impl Identifier for NumericIdentifier {
    fn num(&self) -> Option<u64> {
        Some(self.num)
    }
}

// Direct field access on the concrete type — no Option wrap.
fn main() {
    let id = NumericIdentifier { num: 42 };
    let value: u64 = id.num;                // <- non-nullable access
    let through_trait: Option<u64> = id.num(); // <- still Option<u64> through the trait
}
```

**Rules:**

1. Store the value as `T` (not `Option<T>`) on the concrete child — non-null narrowing
   removes any reason to wrap.
2. The trait `impl` for the child always returns `Some(...)`; the trait itself remains
   `Option<T>`-typed so it stays substitutable with any other implementor.
3. Direct field access on the concrete type is the ergonomic happy path. The trait method
   is for polymorphic / `dyn Identifier` use.
4. When inheriting more than one nullable narrowing in a single child, all `Some(...)`
   wraps live in the same `impl` block — no need for further indirection.

The narrowing rule is currently scoped to nullability only.