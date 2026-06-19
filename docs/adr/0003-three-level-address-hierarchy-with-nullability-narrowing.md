# ADR-0003: Three-level address hierarchy with nullability narrowing

**Status:** Accepted
**Date:** 2026-06-19

## Context

Every entity on a Hiero ledger is referenced through some kind of typed identifier.
Today, the only such identifier in the V3 spec is the flat `Address` in
[`spec/base/ledger.md`](../../spec/base/ledger.md), carrying
`(shard, realm, num, checksum)`. Every concrete reference — token, topic, file,
schedule, account, contract — currently passes through `Address`, with the field name
(`tokenId`, `accountId`, …) doing the disambiguation.

HAPI distinguishes entity kinds with structurally different identifiers, and the V3
surface must eventually reflect that:

1. **`TokenID` / `TopicID` / `FileID` / `ScheduleID`** are pure
   `(shard, realm, num)` triples. There is no other addressing form for them.
2. **`ContractID`** carries `oneof { contractNum, evm_address: bytes }` — a contract
   may be addressed by its Hiero number **or** by its 20-byte EVM address (from
   `CREATE` / `CREATE2` / EIP-1014 / `EthereumTransaction`).
3. **`AccountID`** carries `oneof { accountNum, alias: bytes }` — and the `alias` bytes
   can be either a 20-byte EVM address (HIP-583) or a serialised protobuf `Key`
   (HIP-32).

The V2 Hedera Java SDK exposes one PascalCase type per entity kind (`AccountId`,
`ContractId`, `TokenId`, `TopicId`, `FileId`, `ScheduleId`, `NftId`, `EvmAddress`, …).
V3 has so far modelled all of these as the single `Address`, which is sound for the
four pure-`shard.realm.num` kinds but loses information for `AccountID` / `ContractID`
(no place for the `oneof` alias variant) and is structurally dishonest for
`EvmAddress` (which is not a `shard.realm.num` triple at all).

The first concrete consumers for typed account / contract identifiers will be smart
contract transactions ([`missing-features.md`](../../missing-features.md) §1.4), the
`Key` sum type ([§3.2](../../missing-features.md#32-keys)), and HIP-583 / HIP-1027
EVM-address-related APIs. Settling the address shape **before** those land avoids a
double migration through every spec file that today references an account or contract
by `Address`.

### Forces

1. **No HAPI information loss.** The typed identifier must accept every form HAPI
   accepts for the entity kind. Encoding `AccountID.alias` as opaque `bytes` (as the
   wire does) forces every consumer to length-check at runtime.
2. **Language-agnostic mapping.** The meta-language is mapped to dynamically-typed
   bindings (JavaScript, Python) as well as statically-typed ones (Java, Rust, Swift,
   …). Type-level distinctions must be expressible across that spread; constructs
   that work in one but not others are not viable.
3. **Liskov-clean.** The meta-language guideline allows children to *tighten* parent
   invariants but not to *weaken* them. Any chosen hierarchy must respect this —
   widening an inherited `@@oneOf` would be a Liskov violation in the input
   direction.
4. **No surprise migrations.** 28 spec files import `Address` from `ledger`. Adding
   typed account / contract identifiers will eventually mean touching call sites in
   every one of them; the hierarchy must minimise how often that migration repeats.

## Decision

Adopt a **three-level address hierarchy** in
[`spec/base/ledger.md`](../../spec/base/ledger.md):

```
BaseAddress (abstract)
  ├─ Address (final)
  └─ EvmCapableAddress (abstract)
       ├─ ContractId (final)
       └─ AccountId (final)
```

with the following per-type contracts:

- **`BaseAddress`** is an `abstraction` carrying `shard: uint64`, `realm: uint64`,
  `checksum: string`, `@@nullable num: uint64`, and the three shared methods
  (`validateChecksum`, `toString`, `toStringWithChecksum`). Never instantiated
  directly. `num` is `@@nullable` here because EVM-only `ContractId` / `AccountId`
  instances do not always carry one.
- **`Address`** is a `@@finalType` extending `BaseAddress` that **narrows** the
  inherited `num` slot to non-null via a new `@@override` annotation. Used for
  entities whose HAPI id is purely `(shard, realm, num)`: tokens, topics, files,
  schedules. This is also the typed identifier referenced across the rest of the
  spec today.
- **`EvmCapableAddress`** is an `abstraction` extending `BaseAddress`, adding
  `@@nullable evmAddress: EvmAddress`. Carries **no** `@@oneOf` of its own — each
  concrete child tightens with its own.
- **`ContractId`** is a `@@finalType` extending `EvmCapableAddress`. Structurally
  empty (all fields inherited); carries `@@oneOf(num, evmAddress)`. Mirrors HAPI
  `ContractID`'s wire shape.
- **`AccountId`** is a `@@finalType` extending `EvmCapableAddress`, adding
  `@@nullable alias: bytes` (HIP-32 key-alias slot) and carrying
  `@@oneOf(num, evmAddress, alias)`. **Splits** HAPI's opaque `AccountID.alias: bytes`
  into the typed `evmAddress: EvmAddress` slot (HIP-583 EVM-form) and the
  `alias: bytes` slot (HIP-32 key-form) so the type system, not a length check,
  tells the caller which form is held.
- **`EvmAddress`** is a separate `type` (value object) wrapping 20 raw bytes plus
  parsing factories. NOT a `BaseAddress` descendant — it is not a
  `(shard, realm, num)` triple.

Hoisting `num` to `BaseAddress` requires **one meta-language addition**: a new
`@@override` annotation and a *Narrowing inherited nullability* rule in
[`guidelines/api-guideline.md`](../../guidelines/api-guideline.md). The rule allows
a child to re-declare an inherited `@@nullable` field as non-`@@nullable` provided
it keeps the type and `@@immutable` discipline identical and tags the
re-declaration with `@@override`. The rule is LSP-strengthening (children tighten,
not weaken) — symmetric to but explicitly **not** the same as `@@oneOf` widening,
which would be LSP-weakening and remains forbidden.

The meta-language rule and the address hierarchy are recorded together because the
hierarchy is not buildable in its honest form without the rule. Without `@@override`,
`num` could not be hoisted to `BaseAddress` without either making it `@@nullable`
on `Address` too (sacrificing type safety for tokens / topics / files / schedules —
every caller would have to null-check `tokenId.num` despite the protocol invariant
that it is always set) or duplicating the field declaration in `Address` and
`EvmCapableAddress`.

## Alternatives considered

### A. Keep one flat `Address` for everything

Rejected. `Address` cannot express HAPI's `oneof` for `AccountID` / `ContractID`.
The EVM-alias and key-alias forms have nowhere to live; consumers side-channel
them via ad-hoc separate fields — which the spec already does today on
[`mirror-node-account.md`](../../spec/mirror-node-client/mirror-node-account.md)
line 16 (`evmAddress: string`) and
[`queries-accounts.md`](../../spec/consensus-node-client/queries-accounts.md)
line 52 (`evmAddress: bytes`). HIP-583 auto-create scenarios (no `accountNum`
assigned yet) cannot be represented at all.

### B. Parallel hierarchies — no shared root

Rejected. `AccountId` and `ContractId` would each declare their own `shard`,
`realm`, `checksum`, `num`, `evmAddress`, plus the shared method bodies. `Address`
would stay separate. There would be **no** single base type that polymorphic code
could accept ("any addressable entity on this ledger"). The `(shard, realm)`
invariant — universal across all HAPI entities — would not be visible in the
type system at all. An earlier draft used this shape; it was abandoned for the
readability cost alone, before LSP concerns were considered.

### C. `EvmCapableAddress` with `@@oneOf(num, evmAddress)` on the abstraction

Rejected. If the intermediate abstraction carries `@@oneOf(num, evmAddress)`,
`AccountId` cannot extend it: `AccountId` needs `@@oneOf(num, evmAddress, alias)`,
which would *widen* the parent's two-way constraint to a three-way one. Widening
is a Liskov violation in the input direction — an `AccountId` instance with only
`alias` set would be invalid as an `EvmCapableAddress`, breaking substitution.
We considered extending the meta-language to allow `@@oneOf` widening; that would
have been an LSP-weakening rule. We judged it unsafe and a poor precedent
(see alternative F below). Putting the `@@oneOf` on the concrete children only
avoids the issue.

### D. Sibling hierarchy: `BaseAddress` + `Address` + `EvmCapableAddress`, no `num` hoist

Considered. Three-level structure, but `num` declared on **both** `Address`
(non-null) and `EvmCapableAddress` (nullable). Avoids the meta-language change.
Rejected because the `num` slot is a universal property of HAPI entity addresses
— every kind has one logically, even if some kinds permit it to be absent.
Splitting its declaration hides that universality, invites future inconsistencies
(a third subtype could add `num` with a different annotation by accident), and
denies `BaseAddress` the polymorphism that the universal "has a num slot"
property would otherwise enable.

### E. One PascalCase type per HAPI entity kind (V2 SDK shape)

Considered. `TokenId`, `TopicId`, `FileId`, `ScheduleId` would become separate
types alongside `AccountId` and `ContractId`. Rejected: the four pure-`shard.realm.num`
kinds carry no structural information beyond the address tuple. Wrapping each in
its own type buys compile-time distinguishability only — at the cost of N
identical wrappers, N factory sets, and per-language boilerplate. For a
meta-language that also targets dynamically-typed bindings (JS, Python), the
marginal type-safety gain does not justify the surface area. The field name
(`tokenId`, `topicId`, …) carries the semantic; `Address` carries the structure.
Recorded as a deliberate non-feature in
[`missing-features.md`](../../missing-features.md) §3.1.

### F. Extend the meta-language to allow `@@oneOf` widening

Rejected. Widening (children allow more states than parents) weakens the parent
contract. A consumer holding an `EvmCapableAddress` reference and dispatching on
its `@@oneOf` variants would either crash on the child's extra variant or have
to be defensively coded around the possibility. Languages with closed tagged
unions (Rust enums, TypeScript discriminated unions, Scala sealed traits) would
map this poorly — every consumer would need a default case, or the binding
would have to generate per-subtype converters. Narrowing is the safe direction;
widening is not. The narrowing rule introduced for this ADR is **deliberately
asymmetric** with this rejected alternative — the meta-language now strengthens
but does not weaken in subtypes.

## Why the meta-language change is integral, not optional

The `@@override` rule could be deferred and the hierarchy could be modelled by
field duplication (alternative D above). It is part of *this* decision because:

1. **Honesty of the model.** The `num` slot is universal across entity addresses.
   Splitting its declaration across two siblings hides that universality and
   invites future inconsistencies.
2. **One-time meta-language cost vs. permanent spec cost.** The narrowing rule
   is ~20 lines in `api-guideline.md` plus ~50 lines split across three language
   guides. Field duplication, by contrast, lives in the spec forever and at
   every future review of the address types.
3. **The rule is LSP-strengthening.** Adding a strengthening rule to the
   meta-language is structurally safer than the rejected widening would have
   been. The cost-benefit is asymmetric in favour of adding it now: the
   strengthening direction has applications beyond addresses (any future
   hierarchy with a child that genuinely guarantees more than its parent), and
   its rules are well-defined.

## Consequences

### Positive

- Every HAPI fact about entity addressing is expressible at the type level: the
  `(shard, realm)` universality on `BaseAddress`, the `num`-or-EVM choice on
  `EvmCapableAddress`, the additional key-alias on `AccountId`, the strict
  `(shard, realm, num)` form on `Address`.
- HIP-583 auto-create scenarios (no `num` assigned yet) can be represented
  honestly: `accountId.num == null` is a valid state of an `AccountId`, but
  never of a `tokenId: Address`.
- The Liskov-strengthening direction is now an explicitly available tool in the
  meta-language. Future hierarchies where a child genuinely guarantees more
  than its parent can use the same rule without rediscovering the design.
- All 28 existing spec files that import `Address` from `ledger` continue to
  work unchanged — `Address` is still a concrete type with the same fields and
  the now-stronger non-null `num` contract.
- Polymorphism over `EvmCapableAddress` is available for future APIs (`Key`
  sum-type variants, EVM-related transactions, mirror-node lookups by EVM
  address) that need to accept "account or contract by either form".

### Negative

- The meta-language acquires a new annotation (`@@override`) and a new rule.
  Contributors and reviewers must learn it. The rule applies today only to
  nullability narrowing, but the door is now open — extending it to other
  constraints (e.g. narrowing inherited `@@max`) would require its own
  deliberate ADR.
- Three of the documented target languages need a non-trivial mapping section
  to express nullability narrowing idiomatically. Java pays a small ceremony
  at the boxed/primitive boundary (`@Nullable Long` → `@NonNull Long`, optional
  unboxed accessor); Rust pays a small ceremony for trait+impl shadowing (no
  inheritance to map onto); Go is similar.
- Existing call sites still pass `Address` for account / contract references.
  The migration to typed `AccountId` / `ContractId` is queued in
  [`missing-features.md`](../../missing-features.md) §3.1 as a mechanical
  sweep — a non-zero amount of churn across every spec file that touches
  accounts or contracts.
- The hierarchy adds two abstract types (`BaseAddress`, `EvmCapableAddress`) on
  top of the three concrete ones. Readers of `spec/base/ledger.md` must hold
  three levels in their head rather than one.

### Reversibility

Moderate, with a sharp asymmetry between the two halves of the decision.
Collapsing the hierarchy back to a flat `Address` would either lose the ability
to express HAPI's `oneof` for accounts and contracts (impossible without
re-adding type structure) or accept a permanent regression in fidelity. The
`@@override` annotation itself can be deprecated and replaced with field
duplication if the rule turns out to be a net negative; the hierarchy shape
is the load-bearing decision and effectively a one-way door once consumers
(smart-contract transactions, `Key` sum type, HIP-583 APIs) begin to depend on
the typed `AccountId` / `ContractId` distinctions. Treat the hierarchy shape
as final; treat the meta-language rule as reviewable if a concrete downside
surfaces.

## Sources

- Spec under discussion: [`spec/base/ledger.md`](../../spec/base/ledger.md).
- Meta-language rule and the `@@override` annotation: *Narrowing inherited
  nullability* in
  [`guidelines/api-guideline.md`](../../guidelines/api-guideline.md), with
  per-language mappings in
  [`api-best-practices-java.md`](../../guidelines/api-best-practices-java.md),
  [`api-best-practices-rust.md`](../../guidelines/api-best-practices-rust.md),
  [`api-best-practices-js.md`](../../guidelines/api-best-practices-js.md).
- Tracking entry: [`missing-features.md`](../../missing-features.md) §3.1.
- Existing call sites (28 files importing `Address` from `ledger`): enumerated
  via `grep -rn 'requires {Address} from ledger' spec --include='*.md'`.
- Ad-hoc EVM-address fields in current spec, used as one motivating data point
  for B above: [`spec/mirror-node-client/mirror-node-account.md`](../../spec/mirror-node-client/mirror-node-account.md)
  line 16 (`@@nullable evmAddress: string`),
  [`spec/consensus-node-client/queries-accounts.md`](../../spec/consensus-node-client/queries-accounts.md)
  line 52 (`@@nullable evmAddress: bytes`).
- HAPI proto `AccountID` with `oneof { accountNum: int64, alias: bytes }` and
  the alias semantic ("may be an alias public key, or an EVM address"):
  hiero-ledger/hiero-consensus-node,
  [`hapi/hedera-protobuf-java-api/src/main/proto/services/basic_types.proto`](https://raw.githubusercontent.com/hiero-ledger/hiero-consensus-node/main/hapi/hedera-protobuf-java-api/src/main/proto/services/basic_types.proto),
  lines ~290–330, accessed 2026-06-19.
- HAPI proto `ContractID` with `oneof { contractNum: int64, evm_address: bytes }`
  (20-byte EVM address): same file, lines ~348–385, accessed 2026-06-19.
- HIP-583 "Expand alias support in CryptoCreate & CryptoTransfer Transactions"
  (Final, Standards Track) — formalises 20-byte EVM-address aliases derived
  from ECDSA secp256k1 public keys via Keccak-256 hash; uses the existing
  `AccountID.alias` bytes field; introduces auto-creation of "hollow accounts"
  on transfer to a non-existent EVM-address alias:
  [`HIP/hip-583.md`](https://raw.githubusercontent.com/hashgraph/hedera-improvement-proposal/main/HIP/hip-583.md),
  accessed 2026-06-19.
- HIP-32 "Auto Account Creation" (Final, Standards Track) — public-key-based
  aliases stored as serialised protobuf `Key` bytes in the same `AccountID.alias`
  field, with textual long-form encoding
  `shard.realm.<base32url(Key bytes)>`:
  [`HIP/hip-32.md`](https://raw.githubusercontent.com/hashgraph/hedera-improvement-proposal/main/HIP/hip-32.md),
  accessed 2026-06-19.
- Repository scope and constraints: [`CLAUDE.md`](../../CLAUDE.md).
