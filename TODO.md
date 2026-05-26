# TODO

Open follow-up tasks for the V3 SDK prototyping effort. See [`CLAUDE.md`](CLAUDE.md) for project orientation and
[`guidelines/api-guideline.md`](guidelines/api-guideline.md) for the meta-language.

## Best-practice guides

- [ ] **Document the `$$Self` self-type mapping in every language best-practice guide.**
  The native-token abstraction uses an F-bounded self-type
  (`NativeToken<$$Self extends NativeToken<$$Self, $$Unit>, $$Unit extends NativeTokenUnit>`, see
  [`spec/base/native-token.md`](spec/base/native-token.md)) so that `to(...)` returns the concrete type (e.g. `Hbar`)
  instead of the abstraction. Each guide must describe the idiomatic mapping:
  - Swift: `Self` (protocol) — no explicit self parameter needed
  - Rust: `-> Self` on the trait, with `$$Unit` modeled as an associated type
  - TypeScript: polymorphic `this` return type
  - Java: explicit F-bound (`<SELF extends NativeToken<SELF, U>, U extends NativeTokenUnit>`); wildcard usages become
    `NativeToken<?, ?>`
  - C++: CRTP (`template<class Self, class Unit>`)
  - Go: interface-based mapping (no generics-based self-type)
  - Python: `Self` (PEP 673) / `TypeVar`
  - Also consider adding a dedicated "self type" section to `guidelines/api-guideline.md` so the convention is defined
    once for all languages.

- [ ] **Document the implicit enum `values()` mapping in every language best-practice guide.**
  Every enum implicitly provides `@@static list<EnumName> values()` (now defined in
  [`guidelines/api-guideline.md`](guidelines/api-guideline.md) → Enumerations) and must not declare it explicitly.
  Each guide must describe how this is realized:
  - Java: built-in `values()`
  - Swift: `CaseIterable.allCases`
  - Python: `list(MyEnum)`
  - TypeScript / JavaScript: `Object.values(MyEnum)`
  - Rust: `strum::IntoEnumIterator` or a manually maintained `const` array
  - Go: a maintained slice of the constant values
  - C++: a manually maintained array / generated table
