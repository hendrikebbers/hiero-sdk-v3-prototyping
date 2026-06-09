# ADR-0001: Split the transaction API into `Transaction` and `PackedTransaction`

**Status:** Accepted
**Date:** 2026-06-09

## Context

The transaction API in [`spec/consensus-node-client/transactions.md`](../../spec/consensus-node-client/transactions.md)
must support a wide range of signing and submission flows: a simple operator-driven
"sign-and-execute" path, multi-signature collection across processes or organisations,
out-of-process signing via HSMs and hardware wallets, async server-side signing
pipelines, and audit archival of the exact pre-sign payload.

Three protocol-level facts shape the surface:

1. A signature is computed over the serialized `TransactionBody`. Any change to
   the body after signing silently invalidates the signature; the consensus node
   rejects the transaction with `INVALID_SIGNATURE` at submission time.
2. The `TransactionBody` carries the target node's `nodeAccountID` field. The
   network rejects bodies addressed to a different node (`INVALID_NODE_ACCOUNT`).
   The SDK therefore prepares N distinct body variants for N target nodes, each
   needing its own signature from every signing key.
3. The `TransactionId` (payer + `validStart`) is part of the signed body and
   must exist before signing. `validStart` is bounded by `validDuration` (default
   120 s, max 180 s) — generating it too early shrinks the usable execution
   window.

The V2 Hedera Java SDK collapses build, freeze, sign, and execute into a single
mutable `Transaction` class with a `freeze()` step that gates further mutation.
This works but leaks two recurring failure modes into the API:

- Mutating a frozen `Transaction` after signing fails at execute time, not at
  edit time. The constraint is documented but not type-enforced.
- Whether a given operation is legal depends on a hidden boolean flag (`isFrozen`),
  not on the type of the object.

V3 is greenfield with no backward-compatibility constraint
([`CLAUDE.md`](../../CLAUDE.md#what-this-repository-is)), so the question is
whether to repeat this shape or choose a structurally clearer alternative.

## Decision

Split the lifecycle into two types:

- **`Transaction<$$Receipt>`** — the editable builder. Service-specific
  fields and the transaction-level fields (`maxTransactionFee`, `validDuration`,
  `memo`) live here. No payer, no `TransactionId`, no signatures, no target nodes.
- **`PackedTransaction<$$Receipt, $$Transaction>`** — the wire-ready form. Carries
  an immutable `transactionId`, the list of target `nodes`, and a `nodeSignatures`
  list that grows as signatures are appended. The body itself is byte-stable from
  this point on.

The transition is the explicit method `Transaction.pack(payer, nodes)`. Convenience
methods (`signWithOperator(client)`, `sign(payer, nodes)`,
`sign(payerId, signer, nodes)`) collapse pack and sign into one call for the
common cases.

## Alternatives considered

### A. Keep a single mutable `Transaction` type with an internal `isFrozen` flag (V2 shape)

Rejected. The constraint "body bytes must not change after signing" is enforced
only at runtime, by ad-hoc checks inside individual setters. The same method
(`setMemo`) behaves differently depending on hidden state. Type-level visibility
is lost.

### B. Single type with a `freeze()` method that returns the same type

Rejected for the same reason as A. Even if `freeze()` is offered, the type
signature does not change after the call — the compiler cannot prevent code from
calling `setMemo()` on a frozen transaction. The constraint is still runtime-only.

### C. Single type with a `freeze()` method that returns a new sealed type

This is structurally equivalent to the chosen split, only with different naming.
The decision then becomes a naming question, see *Naming: `pack` vs. `freeze`*
below.

### D. Builder pattern (`TransactionBuilder` → `Transaction`)

Considered. The naming would suggest that `Transaction` is the immutable form
and `TransactionBuilder` is the mutable form. Rejected for two reasons:

1. The mutable form is what every user-facing example and every service-specific
   sub-namespace (`AccountCreateTransaction`, `TransferTransaction`, …) is
   centred on. Renaming the central concept to `…Builder` adds a noun that users
   carry around without it earning its keep.
2. `Transaction` is the well-established noun for "a transaction one constructs
   and submits"; `PackedTransaction` is a precise label for "a transaction
   that has been prepared for the wire". The chosen names match the user's
   mental model better than a generic `Builder` would.

## Naming: `pack` vs. `freeze`

Both names describe the same operation — produce a byte-stable, wire-ready
artefact from an editable builder. `pack` was chosen because:

- **`pack` describes the product.** A packed transaction is something concrete:
  it has been bundled with its target nodes, its transaction id, and the
  encoded body bytes. `PackedTransaction` reads naturally as "the result of
  packing"; the type and the verb reinforce each other.
- **`freeze` describes only what is forbidden afterwards.** It says "you can no
  longer edit", but does not name the thing that now exists. `FrozenTransaction`
  inherits the same drawback — it is defined by what it cannot do.
- **`freeze` is overloaded in language ecosystems.** `Object.freeze` in
  JavaScript, `freeze` in Ruby, and `frozenset` in Python all describe a
  *property* of an existing object (you may no longer mutate it) without
  changing its type. The V3 operation changes the type — the choice of a
  different verb avoids the false analogy.
- **`pack` mirrors the wire metaphor.** The output of `pack(...)` is what
  travels across processes, queues, and serialization boundaries. The V2
  protobuf type that travels on the wire is literally called
  `SignedTransaction`, wrapped in a `TransactionList`. Calling the SDK's
  pre-wire representation `PackedTransaction` matches that mental model and
  pairs cleanly with `toBytes()`/`fromBytes()`.

## Why a single type cannot cleanly model both phases

A single mutable type would need every field that exists on either phase
(`payer`, `nodeSignatures`, `transactionId`, `nodes`, plus all the editable body
fields) and would need runtime guards on every setter and every signature
operation to keep them consistent. Concretely:

- `setMemo(...)` would have to throw at runtime if any signature has been
  collected — invariant maintained by documentation, not by the type.
- `addSignature(...)` would have to assert that the body is "ready" (payer
  set, nodes set, id generated) — again, runtime-only.
- `toBytes()` would have to behave differently depending on whether signatures
  have been collected (an empty `SignatureMap` vs. a populated one) — a single
  method with two semantically distinct return values.
- The static `fromBytes(...)` factory would have to decide whether the loaded
  bytes are still editable. They never are — once on the wire, the body is
  fixed — but the type would not express this.

With two types, every one of these concerns becomes a static check:

- `setMemo(...)` only exists on `Transaction`. The compiler refuses to call it
  on a `PackedTransaction`.
- `addSignatures(...)` only exists on `PackedTransaction`. It cannot run on
  something that has not been packed.
- `toBytes()` and `fromBytes(...)` only ever operate on `PackedTransaction`.
  There is no ambiguity about what was serialized.
- The state machine `build → pack → sign → execute` is encoded in return types,
  not in defensive runtime checks.

## Consequences

### Positive

- The "body must not change after signing" constraint is enforced by the type
  system rather than by runtime assertions.
- Out-of-process and air-gap-adjacent flows (HSM, hardware wallets, multi-party
  signing, async pipelines) are first-class — `Transaction.pack(...)` produces
  a transportable artefact without any signature being involved.
- Convenience methods (`signWithOperator`, `signWithOperatorAndExecute`) keep
  the 95 % path one method call long. The split adds no friction for simple
  cases.
- The on-the-wire vocabulary (`PackedTransaction`, `toBytes`, `fromBytes`)
  matches what V2 already exposes via protobuf — fewer surprises when reasoning
  about the wire layer.

### Negative

- Two types are more surface to learn, document, and translate to every target
  language. Language guides (`api-best-practices-{java,rust,js}.md`) must each
  show the split idiomatically.
- The single generic parameter `$$Transaction extends Transaction<$$Receipt>`
  threads through every `PackedTransaction` method signature in the spec. The
  visual noise is real, even if the type-system payoff is solid.
- A user familiar with V2 must unlearn the "single `Transaction` class with
  `freeze()`" pattern. Migration documentation must call this out.
- `Transaction.pack(...)` followed by `PackedTransaction.sign(...)` is two
  steps where V2 has one — the verbose form is only used in advanced cases,
  but it exists.

### Reversibility

Low cost to evolve, high cost to undo. Adding overloads on `pack(...)` or new
signing helpers is purely additive. Collapsing the two types back into one
would break every concrete `Transaction` subtype and every Mirror-Node or
Enterprise-layer consumer that holds onto a `PackedTransaction` reference.
Treat this as a one-way decision.

## Sources

- Spec under discussion: [`spec/consensus-node-client/transactions.md`](../../spec/consensus-node-client/transactions.md)
- Sibling types referenced: [`spec/consensus-node-client/client.md`](../../spec/consensus-node-client/client.md) for `NodeSignature`, `TransactionSigner`, `Account`, `HieroClient`.
- Repository scope and constraints: [`CLAUDE.md`](../../CLAUDE.md).
- Hedera protobuf definition of `TransactionBody` and `TransactionID` (fields, `nodeAccountID` semantics, `validDuration` bounds): [hashgraph/hedera-protobufs](https://github.com/hashgraph/hedera-protobufs).
- Hedera Java SDK V2 `Transaction.freezeWith(...)` behaviour (TransactionId auto-generation from operator, fixed node list after freeze): [hiero-ledger/hiero-sdk-java](https://github.com/hiero-ledger/hiero-sdk-java/blob/main/sdk/src/main/java/com/hedera/hashgraph/sdk/Transaction.java).
- V2 lifecycle documentation cited at decision time: [Hedera SDK Client guide](https://docs.hedera.com/hedera/sdks-and-apis/sdks/client), [Hedera SDK Transactions overview](https://docs.hedera.com/hedera/sdks-and-apis/sdks/transactions).
