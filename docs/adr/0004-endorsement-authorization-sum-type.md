# ADR-0004: Model HAPI authorization keys as an `Endorsement` sum type

**Status:** Proposed
**Date:** 2026-06-22

## Context

Every entity on a Hiero/Hedera consensus node is guarded by an **authorization requirement** —
the structure that decides who or what may act on it (sign for it, update it, mint, …). HAPI
models this as a protobuf message it calls `Key` — an overloaded name this ADR sets out to
replace (the term is the whole problem; see *Decision*). Verified against the protocol
definition, that message is a recursive **sum type** whose variants reduce to three conceptual
kinds:

- a **public key** — `ed25519` or `ECDSA_secp256k1` (plus the deprecated `RSA_3072` / `ECDSA_384`);
  these are always *public* keys, never private;
- a **contract** — `contractID`, or `delegatable_contract_id` (authority also usable via
  `delegatecall`);
- a **composition** — `keyList` (`repeated Key`, all-must-sign / n-of-n) or `thresholdKey`
  (`{ threshold: uint32, keys: KeyList }`, m-of-n). The composition is recursive: an account can
be guarded by "Alice **and** (2-of-3 of {Bob, Carol, contract})".

### The placeholder problem

The capability is not in question — V3 will implement it. The problem is that no upstream model is
worth adopting wholesale. HAPI expresses it as the wire-oriented `Key` `oneof` above (in-band
clear-sentinel, deprecated algorithm variants and all); the V2 SDKs mirror that shape with an
everything-is-a-`Key` inheritance hierarchy — in the Hiero Java SDK, `Key` is an abstract class
whose subtypes include the public keys, the identifier types `ContractId` / `EvmAddress`, and
`KeyList`, folding key material, identifiers, and composition into one type (the very conflation
this ADR sets out to avoid). V3's specs have so far declined to commit and stand in with a
placeholder: roughly 57 key fields are typed as `PublicKey` or
`list<PublicKey>` (`transactions-tokens.md` 8 keys, account `key`,
file `keys`, topic `adminKey`/`submitKey`, and their read-side mirrors in `queries-*` and
`mirror-node-*`). `missing-features.md` §3.2 records `Key` / `KeyList` / `ThresholdKey` as
missing and the prerequisite for the rest. The placeholder loses three real capabilities:
contract-controlled authority (`contractID` keys — e.g. a token whose `supplyKey` is a smart
contract), m-of-n custody (`thresholdKey`), and nesting. A flat `set<PublicKey>` cannot express
any of them, because the leaf is a `Key`, not a public key.

### Forces

- **Naming collision.** A `keys.Key` abstraction already exists, but it models *cryptographic
  material* (`bytes`, `algorithm`, `type`), and `PublicKey`/`PrivateKey extends Key`
  (`keys.md:43,67,75`). HAPI's `Key` is a logical *authorization* construct, not material.
- **`PrivateKey` is never authorization.** Only public information ever appears in a HAPI key.
  A private key must be structurally excluded from the authorization hierarchy.
- **Layering.** `keys` is dependency-free; `ledger` depends only on `nativeToken`
  (`ledger.md:104`). The authorization type references both `PublicKey` (keys) and `ContractId`
  (ledger). The dependency arrows must not invert or cycle.
- **Sealing.** The set of variants is closed and is both produced (info queries) and consumed
  (wallets rendering "2-of-3", "am I a required signer", key reuse on update). Consumers need
  exhaustive, compiler-checked destructuring.
- **In-band sentinels.** HIP-540 clears a key with an empty `KeyList` sentinel — overloading the
  data type to signal an *operation*. The behaviour is defined, but the value-vs-operation
  distinction is not modelled. V3 has no wire-modelling-compat obligation and should not inherit
  the overload.

## Decision

We introduce a dedicated **`endorsement`** namespace defining a sealed `Endorsement` sum type
that replaces the `PublicKey` placeholder for authorization keys. (**`Authority`** is the
documented alternative name for the root type — equally well-scoped; the choice between the two
is a coinage preference, not a structural one.) The design is governed by one principle that drove
every sub-decision:

> **No in-band sentinels.** Each distinct meaning is its own type or variant — never a magic
> value or an overloaded `null`.

```
namespace endorsement
requires {PublicKey} from keys
requires {ContractId} from ledger

// Pure data sum type — no behavior methods. Immutable value type with structural
// equality (provided idiomatically by each language binding: Java records,
// Rust derive(PartialEq), etc.). Two Endorsements are equal iff their trees match.
@@sealed abstraction Endorsement {}

@@finalType PublicKeyEndorsement extends Endorsement { @@immutable publicKey: PublicKey }

@@finalType ContractEndorsement extends Endorsement {
    @@immutable contractId: ContractId
    @@immutable @@default(false) delegatable: bool   // false → ContractID, true → DelegatableContractID
}

@@finalType EndorsementList extends Endorsement {
    @@immutable @@minLength(1) children: list<Endorsement>
    @@immutable @@min(1) threshold: int32            // always set; invariant 1 ≤ threshold ≤ children.size()
}

// construction (blessed path; leaves are wrapped internally):
@@static Endorsement of(children: Endorsement...)             // n-of-n: threshold = children.size()
@@static Endorsement of(threshold: int32, children: Endorsement...)   // m-of-n
@@static Endorsement of(publicKey: PublicKey)                 // PublicKeyEndorsement
@@static Endorsement ofContract(contractId: ContractId)       // plain ContractID
@@static Endorsement ofDelegatable(contractId: ContractId)    // DelegatableContractID
```

Concretely we will:

- **Use composition, not inheritance.** Variants *wrap* `PublicKey` and `ContractId` rather than
  those types extending `Endorsement`. This is the load-bearing choice: it keeps the dependency
  one-directional (`endorsement → {keys, ledger}`), so `keys`/`ledger` stay foundational and know
  nothing about `endorsement`, and it lets all variants be co-located for real compile-time
  sealing. `PrivateKey` simply has no wrapper, so it is structurally never an `Endorsement`.
- **Model the composite as a single type with a mandatory `threshold`.** n-of-n is
  `threshold == children.size()`, not a sentinel. Factories hide the redundant threshold for the
  common all-of case.
- **Rely on value-type structural equality**, provided idiomatically by each language binding
  (records / `derive(PartialEq)`), rather than declaring an explicit `equals` method. This
  satisfies the read-side need ("is this the same key I'm about to write?") without adding API
  surface, consistent with the rest of the specs. `Endorsement` carries **no behavior methods** —
  it is a pure data sum type; any flatten/inspect convenience (e.g. collecting leaf public keys)
  is derivable by pattern-matching and is deferred until a concrete need appears.
- **Treat `@@sealed` as forthcoming.** The spec uses it now; a separate follow-up adds it to the
  meta-language with the per-language mappings (Java `sealed`, Rust `enum`, TS discriminated
  union).

### Rejected alternatives

- **`set<PublicKey>` / `list<PublicKey>` (the status quo).** Cannot express contract keys,
  thresholds, or nesting. This is exactly the placeholder we are removing.
- **Reuse the name `Key`.** Forces the absurd statement "a `PrivateKey` is not a `Key`" (because
  private keys must be excluded), and collides with the existing cryptographic `keys.Key`.
  `Allowance` was also rejected — it is already a distinct HAPI concept (HIP-336 spend approval).
  `Endorsement` is a deliberate **coinage**, not an industry-standard term: collision-free, and
  precise (a leaf "endorses"; covers both signature and contract-execution satisfaction). No
  major ledger names the account-authorization construct "endorsement" — Bitcoin and Solana say
  "multisig" / "m-of-n", Safe on Ethereum says "owners" + "threshold", and Hedera v2 itself uses
  `Key` / `KeyList` / `ThresholdKey`. Those terms all name only the *m-of-n composite*, so none
  can name a root that must also cover a single key or a contract leaf. The closest existing term
  is Hyperledger Fabric's "endorsement policy" (`AND` / `OR` / `OutOf` over principals) — but that
  governs which peers endorse a *transaction* under the Execute-Order-Validate model, a
  structurally identical m-of-n signature policy in a different domain. It is an analogy, not a
  precedent. `Authority` is the documented runner-up coinage (see *Decision*).
- **`MultiSig` (or any term naming only the m-of-n case) as the root name.** Most discoverable —
  it is the term a developer actually searches for. Rejected because it is *semantically false for
  the root*: the root holds three things — a single key (the **most common** case), a contract (no
  signature at all), and the m-of-n composite — and "MultiSig" is true for only the last. Naming a
  union after its most complex special case is a category error: it is like naming a `Number` type
  `Fraction` because fractions are the interesting case, then having to call `5` a `Fraction`.
  "MultiSig" describes 1 of 3 variants, and not the most common one. Discoverability is recovered
  the right way — by putting the `multisig` / `m-of-n` keyword on the **factory** (e.g. a
  `multiSig(...)` / `threshold(...)` factory) and in the type's doc text — so a search lands on the
  factory without forcing the root type to lie.
- **Inheritance (`PublicKey`/`ContractId extends Endorsement` directly) — a genuine alternative,
  close call.** This is the other good option, not a weak one. It reads better (a `PublicKey`
  *is* an `Endorsement`; no wrapper, no `.publicKey` indirection on read, bare-type usability
  preserved) and is how the Hedera v2 SDK is shaped (`PublicKey extends Key`). Its cost is sealing:
  a *compile-time* `@@sealed` root must name its permitted subtypes, but those subtypes live in
  `keys`/`ledger`, which would then depend on `endorsement` → dependency cycle. Inheritance is
  therefore only viable if `@@sealed` is downgraded to a **runtime-only** convention (closure
  checked by `instanceof`, no compiler exhaustiveness). We chose composition because we valued
  compile-time exhaustive matching on the read side over bare-type ergonomics — but the trade is
  close, and if the meta-language cannot deliver compile-time `@@sealed`, or if bare-type
  usability proves more valuable in practice, this inheritance variant is the natural fallback and
  should be reconsidered.
- **Two composite types (`EndorsementList` + `ThresholdEndorsement`).** Considered to avoid a
  nullable threshold; rejected because a *mandatory* threshold removes the sentinel without
  splitting the type.
- **Folded composite with `@@nullable threshold` (null ⇒ all).** Rejected: that `null` is itself
  an in-band sentinel, the very smell this ADR forbids.
- **`EmptyEndorsement` variant for key clearing.** Rejected: re-introduces the value-vs-operation
  overload as a type. On read, "no key / immutable" is already the `@@nullable` field being
  `null`; on write, clearing is an *operation* and belongs to a future `KeyUpdate` type on the
  update transactions, not to the value set.

## Consequences

### Positive
- The ~57 placeholder fields can be retyped to `Endorsement`, unlocking contract-controlled
  authority, m-of-n custody, and nesting — the capabilities HAPI actually has.
- Compile-time exhaustive pattern matching on the read side (sealed, co-located variants).
- `keys` and `ledger` remain foundational and dependency-light; no cycle, no inversion.
- The "no in-band sentinels" principle makes the value set honest: no `null`/empty/magic value
  carries hidden meaning.

### Negative
- **Bare-type usability is surrendered.** A raw `PublicKey`/`ContractId` cannot be passed where an
  `Endorsement` is expected; callers go through factories, and the read side pays a
  `.publicKey`/`.contractId` indirection hop.
- **The meta-language gains a dependency on a not-yet-existing feature (`@@sealed`).** The spec is
  written against a primitive that must still be defined and mapped per language.
- **Construction is not hermetically factory-only.** Public variants (needed for matching) are
  technically directly constructible; factories are the documented/future-proof path, not an
  enforced one.
- **A minor wire round-trip edge:** a redundant on-chain `ThresholdKey(n, n-children)`
  canonicalizes to the `KeyList` wire form on write-back — semantically identical, different
  bytes.

### Follow-ups
- Add `@@sealed` to the meta-language (`api-guideline.md`) with Java/Rust/TS mappings — prerequisite.
- Separate ADR for **`KeyUpdate`** (the three-state Unchanged / SetKey / ClearKey model for
  clearing keys on update transactions), which maps internally to the HIP-540 sentinel.
- Separate meta-language analysis: when is a type an interface-with-hidden-impl vs. a direct
  record — currently underspecified, and it determines whether factory-only construction can be
  enforced.
- Retype the placeholder fields across `transactions-*`, `queries-*`, and `mirror-node-*` and
  update `missing-features.md` §3.2 once `@@sealed` lands. **Revisit this decision** — falling back
  to the inheritance variant (`PublicKey`/`ContractId extends Endorsement`, runtime-only sealing) —
  if compile-time `@@sealed` proves unworkable in the meta-language, or if profiling shows the
  bare-type/indirection cost is unacceptable in practice.

---

**References:**
- HAPI `Key` / `KeyList` / `ThresholdKey` definitions — `hashgraph/hedera-protobufs`,
  `services/basic_types.proto`
  (https://raw.githubusercontent.com/hashgraph/hedera-protobufs/main/services/basic_types.proto),
  accessed 2026-06-22.
- HIP-540 empty-`KeyList` clear-key sentinel —
  https://raw.githubusercontent.com/hashgraph/hedera-improvement-proposal/main/HIP/hip-540.md,
  accessed 2026-06-22.
- HIP-336 (Allowances, the conflicting use of "Allowance") — Hedera HIP registry.
- V2 SDK shape — Hiero Java SDK `Key` is an abstract class with `PublicKey` (ED25519/ECDSA),
  `ContractId`, `DelegateContractId`, `EvmAddress`, and `KeyList` as subtypes —
  `hiero-ledger/hiero-sdk-java` `sdk/.../com/hedera/hashgraph/sdk/Key.java`, accessed 2026-06-22.
- Comparative authorization terminology (accessed 2026-06-22): Hyperledger Fabric "endorsement
  policy" / `AND`/`OR`/`OutOf` over MSP principals, governing *transaction* endorsement by peers —
  https://hyperledger-fabric.readthedocs.io/en/latest/endorsement-policies.html; Solana SPL Token
  "Multisig" / "M of N multisignatures" — https://www.solana-program.com/docs/token; Safe
  (Ethereum) "owners" + "threshold" — https://docs.safe.global. No ledger uses "endorsement" for
  account-authorization keys; "Endorsement" is a coinage.
- Existing cryptographic `Key` abstraction: `spec/base/keys.md:43,67,75`.
- Namespace dependencies: `spec/base/keys.md` (no `requires`), `spec/base/ledger.md:104`;
  `ContractId` at `spec/base/ledger.md:38`.
- Meta-language: multiple inheritance `guidelines/api-guideline.md:167`; abstraction =
  class-or-interface `guidelines/api-guideline.md:154`; `@@maxLength` on collections
  `spec/consensus-node-client/transactions-tokens.md:219,256`.
- Placeholder + missing-type tracking: `missing-features.md:339-341` (and §3.2); HIP-540 sentinel
  noted in-repo at `spec/consensus-node-client/transactions-tokens.md:469`.
- Related: `docs/adr/0003-three-level-address-hierarchy-with-nullability-narrowing.md` (the
  `ContractId` identifier this type wraps).
