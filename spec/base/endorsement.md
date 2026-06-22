# Endorsement API

This namespace defines the **authorization model** for the SDK: the structure that decides *who or
what may act* on an entity (sign for it, update it, mint, …). It is the V3 replacement for the
`PublicKey` / `list<PublicKey>` placeholder used across the transaction, query, and mirror-node
specs, and corresponds to HAPI's `Key` message. The design and the alternatives it rejects are
recorded in [ADR-0004](../../docs/adr/0004-endorsement-authorization-sum-type.md).

## Description

An `Endorsement` is an immutable, recursive **sum type** — an *authorization requirement* that is
satisfied in one of three ways. It is also known as a **multi-signature (m-of-n)** structure when
composed:

- **`PublicKeyEndorsement`** — satisfied by a signature from a single public key. The leaf wraps a
  [`PublicKey`](keys.md); it deliberately *wraps* rather than *is* a `PublicKey` (see below).
- **`ContractEndorsement`** — satisfied not by a signature but by the named smart **contract**
  being the active executing frame. `delegatable = false` maps to HAPI's `ContractID` (the
  contract must be the direct caller); `delegatable = true` maps to `DelegatableContractID`
  (authority may also be exercised via `delegatecall`). This is how, e.g., a token's `supplyKey`
  can be a contract so that only that contract's logic may mint.
- **`EndorsementList`** — a composition of child endorsements with a `threshold`: at least
  `threshold` of the `children` must be satisfied. **n-of-n** ("all must sign") is simply
  `threshold == children.size()`; **m-of-n** is any smaller threshold. Children are themselves
  `Endorsement`s, so the structure is **recursive**: an account can be guarded by
  "Alice **and** (2-of-3 of {Bob, Carol, contract})".

### Design rules (see ADR-0004)

- **Composition, not inheritance.** The leaves *wrap* `PublicKey` and `ContractId` rather than
  those types extending `Endorsement`. This keeps the dependency one-directional
  (`endorsement → {keys, ledger}`) and lets every variant be co-located so the type can be
  `@@sealed`. The cost, accepted deliberately, is that a bare `PublicKey` / `ContractId` cannot be
  passed where an `Endorsement` is expected — callers go through the factories, and the read side
  reads `.publicKey` / `.contractId` off the matched variant.
- **A private key is never an `Endorsement`.** Only public information appears in an authorization
  requirement; [`PrivateKey`](keys.md) has no wrapper here, so it is structurally excluded.
- **No in-band sentinels.** Every distinct meaning is its own type/variant — never a magic value
  or overloaded `null`. `threshold` is therefore always set and always meaningful (n-of-n is a
  real value, not `null`); an empty `EndorsementList` is forbidden (`@@minLength(1)`). "Remove this
  key" is an *operation* on the write side, not an `Endorsement` value, and is modelled separately
  (a future `KeyUpdate` type — see ADR-0004 follow-ups), not by an empty list.
- **Construction goes through the factories** (`of` / `ofContract` / `ofDelegatable`). They are the
  documented, forward-compatible path; the variants are public so that a received `Endorsement`
  can be destructured by pattern matching.

## API Schema

```
namespace endorsement
requires {PublicKey} from keys
requires {ContractId} from ledger

// Authorization requirement (HAPI `Key`). Pure data sum type — no behavior methods. Immutable
// value type with structural equality (provided idiomatically per language: Java records,
// Rust derive(PartialEq), …): two Endorsements are equal iff their trees match.
//
// @@sealed closes the set of variants so consumers can match exhaustively. The annotation is not
// yet part of the meta-language — it is introduced together with this namespace; see ADR-0004.
@@sealed(PublicKeyEndorsement, ContractEndorsement, EndorsementList) abstraction Endorsement {}

// Leaf: satisfied by a signature from this public key.
@@finalType
PublicKeyEndorsement extends Endorsement {
    @@immutable publicKey: PublicKey
}

// Leaf: satisfied by the named contract being the active executing frame (no signature).
@@finalType
ContractEndorsement extends Endorsement {
    @@immutable contractId: ContractId
    @@immutable @@default(false) delegatable: bool   // false → ContractID; true → DelegatableContractID
}

// Composition: at least `threshold` of `children` must be satisfied.
// n-of-n is `threshold == children.size()`; m-of-n is any smaller threshold.
@@finalType
EndorsementList extends Endorsement {
    @@immutable @@minLength(1) children: list<Endorsement>   // never empty
    @@immutable @@min(1) threshold: int32                    // invariant: 1 ≤ threshold ≤ children.size()
}

// --- factories (the blessed construction path; leaves are wrapped internally) ---

// n-of-n composition ("all must sign"): threshold is set to children.size().
@@static Endorsement of(children: Endorsement...)

// m-of-n composition (multi-signature): at least `threshold` of `children` must be satisfied.
@@static Endorsement of(threshold: int32, children: Endorsement...)

// Single-key leaf.
@@static Endorsement of(publicKey: PublicKey)

// Plain contract leaf (HAPI ContractID).
@@static Endorsement ofContract(contractId: ContractId)

// Delegatable contract leaf (HAPI DelegatableContractID): authority usable via delegatecall.
@@static Endorsement ofDelegatable(contractId: ContractId)
```

## Examples

### Build endorsements

```
PublicKey alice = ...;
PublicKey bob   = ...;
PublicKey carol = ...;
ContractId dao  = ...;

// single key
Endorsement single = Endorsement.of(alice);

// all-of (3-of-3)
Endorsement allOf = Endorsement.of(
    Endorsement.of(alice), Endorsement.of(bob), Endorsement.of(carol));

// 2-of-3 multi-signature
Endorsement twoOfThree = Endorsement.of(2,
    Endorsement.of(alice), Endorsement.of(bob), Endorsement.of(carol));

// nested: Alice AND (2-of-3 of {Bob, Carol, DAO contract})
Endorsement nested = Endorsement.of(
    Endorsement.of(alice),
    Endorsement.of(2,
        Endorsement.of(bob),
        Endorsement.of(carol),
        Endorsement.ofContract(dao)));
```

### Inspect a received endorsement (read side)

```
// `key` comes typed as Endorsement from e.g. an account / token info query.
String describe(Endorsement e) {
    switch (e) {                                          // exhaustive via @@sealed
        case PublicKeyEndorsement k         -> return "key " + k.publicKey.toString(...);
        case ContractEndorsement c          -> return (c.delegatable ? "contract* " : "contract ")
                                                       + c.contractId.toString();
        case EndorsementList l              -> return l.threshold + "-of-" + l.children.size()
                                                       + " [" + l.children.map(describe).join(", ") + "]";
    }
}
```

## Questions & Comments

- **`@@sealed` is forthcoming.** This namespace is written against it; a separate follow-up adds
  `@@sealed` to [`api-guideline.md`](../../guidelines/api-guideline.md) with the per-language
  mappings (Java `sealed`, Rust `enum`, TypeScript discriminated union). Until then the closed set
  is a convention. See [ADR-0004](../../docs/adr/0004-endorsement-authorization-sum-type.md).

- **Distinct from `keys.Key`.** [`keys.Key`](keys.md) models cryptographic *material* (`bytes`,
  `algorithm`, `type`); `Endorsement` is the logical *authorization* construct. They are separate
  hierarchies on purpose — `PublicKeyEndorsement` wraps a `keys.PublicKey`.

- **Name is a coinage; `Authority` is the documented alternative.** No major ledger names the
  account-authorization construct "endorsement" (Bitcoin/Solana: "multisig"; Safe: "owners" +
  "threshold"; HAPI/V2: `Key`). The rationale and the `Authority` alternative are in ADR-0004.

- **Building m-of-n from public keys is verbose** (`of(2, of(alice), of(bob), of(carol))`) because
  bare-type usability was surrendered. Convenience overloads taking `PublicKey...` directly
  (`of(2, alice, bob, carol)`) could reduce this, but a single-arg `PublicKey` would be ambiguous
  with the `of(publicKey)` leaf factory, so they are deferred until the ergonomics are judged
  against real usage.

- **Factory discoverability.** A developer looking for multi-signature support searches "multisig"
  / "threshold", not "endorsement". The keyword is carried in the doc text and on the m-of-n
  factory comment; renaming `of(threshold, …)` to an explicit `multiSig(…)` / `threshold(…)`
  factory is an open option (ADR-0004 rejects naming the *root type* `MultiSig`, not naming a
  factory).

- **Key clearing is not modelled here.** Removing a key (HIP-540, signalled on the wire by an empty
  `KeyList`) is a write-side *operation*, not an `Endorsement` value. It is deferred to a future
  `KeyUpdate` type on the update transactions — see ADR-0004.

- **Wire round-trip edge.** A redundant on-chain `ThresholdKey(n, n-children)` read back as an
  `EndorsementList` with `threshold == children.size()` canonicalises to the `KeyList` wire form on
  write-back — semantically identical, different bytes.
