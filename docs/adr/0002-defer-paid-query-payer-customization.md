# ADR-0002: Defer payer customization for paid queries

**Status:** Accepted
**Date:** 2026-06-10

## Context

`PaidQuery<$$Result>` (defined in
[`spec/consensus-node-client/queries.md`](../../spec/consensus-node-client/queries.md))
attaches a payment transaction to every consensus-node query. In the current revision the
payer is fixed: the SDK builds and signs a `CryptoTransfer` from the client's operator
account to the chosen consensus node.

Several real-world workflows want a payer that is **not** the operator:

- **Paymaster patterns** — a dApp service funds reads on behalf of end users who hold no
  native-token balance.
- **Custodial / multi-tenant backends** — one service-level account pays for all reads;
  per-customer billing is tracked separately in the application layer.
- **Audit accounts** — an auditor's account funds their own queries against a treasury they
  inspect.
- **Bots / indexers** — periodic crawlers with budgets tracked independently from the
  primary operator.

An earlier revision of the spec briefly carried a `@@nullable payer: Account` field on
`PaidQuery`, defaulting to `client.operator`. It was removed before this ADR was written.

## Decision

Defer configurable payer for `PaidQuery`. The client's operator pays for every paid query
in this revision. Callers that need a non-operator payer use the **workaround** of
instantiating a separate `HieroClient` whose operator is the desired payer.

When configurable payer is later added, it must be designed alongside the external-signer
case (HSM, hardware wallet, browser wallet, KMS), not as an in-memory-only `Account` field.

## Alternatives considered

### A. `@@nullable payer: Account` on `PaidQuery`

Same shape as `Transaction.pack(payer: Account, nodes)`. `Account` carries `accountId` plus
an in-memory `privateKey` (see [`client.md`](../../spec/consensus-node-client/client.md)),
so the SDK can sign the `CryptoTransfer` payment without further input.

**Rejected.** `Account` requires the payer's private key to be in process memory. That is
the minority case in production: any non-trivial deployment (CloudHSM, hardware wallets,
WalletConnect dApps, FIPS-certified custody) signs externally and cannot supply a usable
`PrivateKey`. Introducing this variant first would shape the API around the simplest
setting while leaving the dominant production setting unsolved, and would risk a
non-orthogonal extension when the external-signer variant is added later.

### B. `@@nullable payerId: Address` + `@@nullable payerSigner: TransactionSigner`

Mirrors `Transaction.sign(payerId: Address, signer: TransactionSigner, nodes)`. Decouples
the payer's identity from the signing mechanism, so HSMs, hardware wallets, and remote
signers slot in naturally via `TransactionSigner`.

**Rejected for now.** Structurally the most honest variant — and the most likely shape if
this feature is reintroduced — but specifying it requires also deciding:

- How `submit(client)` interacts with the payer signer (stateful field on the query versus a
  `submit(client, payerSigner)` overload).
- Whether `getCost(client)` needs the payer signer too, or if cost discovery uses the
  operator regardless of who eventually pays.
- How the envelope (`PaidQueryResponse`) reports the actual payer when it differs from the
  operator (`paidBy` field, currently not present).

Without a concrete use-case driving these answers, choosing now risks locking in defaults
that the first real consumer will want to revisit. Deferring buys time to design once with
evidence.

### C. Client-level payment callback / signer on `HieroClient`

Move the payer concern out of `PaidQuery` entirely and onto `HieroClient`: extend the client
with an optional callback (or `TransactionSigner` companion) dedicated to paid-query
payments. All `PaidQuery.submit(client)` calls funnel through the callback; if unset, the
operator pays as today. Two shapes were considered:

- **Build-and-sign callback** — `(queryCost, targetNode) -> SignedCryptoTransfer`. The
  callback owns the entire payment transaction, including which account is debited.
- **Sign-only callback with a configured payer id** — the SDK builds the `CryptoTransfer`
  (it knows the node and quoted cost), the callback signs it. Requires
  `queryPayerAccountId: Address` alongside the signer so the body can be built before
  signing.

This mirrors the existing `transactionSigner: TransactionSigner` slot on `HieroClient`
([`client.md`](../../spec/consensus-node-client/client.md)), which V3 already uses for the
operator's signing role. The same pattern, extended to a second role: an additional
payment-only signer.

**Pros**

- Single configuration point — set once on the client, applies to every paid query.
- Zero new surface on `PaidQuery`. The query stays minimal.
- Composes naturally with `TransactionSigner`: HSMs, hardware wallets, KMS slot in without
  further API design.
- Honest about the architectural layer — "who pays for queries" is a deployment-time
  concern (one configuration per service instance), not a per-call concern.

**Cons**

- No per-query payer override on a single client. If two sponsors must pay for different
  queries within one application, you still need two clients — equivalent to
  Alternative D's workaround, just at a different layer.
- Adds a third signing role to reason about on `HieroClient` (operator transactions,
  operator queries, payment-only payer). Each role having a separate signer is honest but
  also more cognitive surface.
- Leaves the choice between client-level (C) and per-query (B) open. Picking C now would
  preclude B without strong evidence that per-query overrides are never needed.

**Rejected for now** for the same reason as B: without a concrete consumer driving the
trade-off (one global query payer per deployment versus per-query overrides), choosing
shape commits to a default that may not match real demand. If C ends up being the right
shape, it can be added later as a `@@nullable` field pair on `HieroClient` without breaking
existing callers — fully additive.

### D. Workaround — separate `HieroClient` per payer (chosen interim solution)

Today's API supports this without any change. A caller that wants the sponsor account to
pay simply constructs a second `HieroClient` whose operator is the sponsor:

```
HieroClient sponsorClient = createClient(networkSettings, sponsorAccount);
PaidQueryResponse<AccountInfo> response = new AccountInfoQuery()
    .accountId(endUserId)
    .maxQueryPayment(NativeToken.of(...))
    .submit(sponsorClient);
```

Costs: an extra client object, the sponsor key must be loadable as an `Account` (same
in-memory-key limitation as Alternative A — workaround D is only available when the sponsor
key is usable in-process). Benefits: zero spec changes, no premature commitment to a payer
API.

## Consequences

### Positive

- The `PaidQuery` surface stays minimal — three fields/methods (`maxQueryPayment`,
  `getCost`, the inherited `submit`) instead of four. Easier to learn, easier to translate
  to every target language.
- No API commitment locks the eventual external-signer design. When the feature comes back,
  it can be designed coherently with HSM and wallet integrations rather than fitted around
  a pre-existing `Account` field.
- Avoids exposing an in-memory-only abstraction (`payer: Account`) that would have implied
  production-readiness in settings where it does not actually work.

### Negative

- Sponsored / paymaster patterns require the workaround (Alternative D) today. This is
  acceptable while V3 has no shipping consumers but will need a proper solution before
  paymaster-style dApps adopt V3.
- A small inconsistency with `Transaction.pack(payer, nodes)`: transactions accept a custom
  payer at the type level, paid queries do not. Documented in `queries.md` so it is not a
  silent gap.
- The external-signer variant (Alternative B) and the client-level callback (Alternative C)
  both remain design work for a later revision. Either can address the HSM / wallet case;
  the choice between per-query and per-client granularity is still open.

### Reversibility

Fully additive when revisited. Reintroducing payer customization — as a `payer` field on
`PaidQuery` (Alternatives A or B) or as a callback / signer slot on `HieroClient`
(Alternative C) — does not break existing callers; the operator default remains the no-op
case in every shape. Treat this ADR as a deliberate deferral, not a permanent exclusion.

## Sources

- Spec under discussion:
  [`spec/consensus-node-client/queries.md`](../../spec/consensus-node-client/queries.md)
  (`PaidQuery` abstraction, payment-model section).
- Comparison point on the transaction side:
  [`spec/consensus-node-client/transactions.md`](../../spec/consensus-node-client/transactions.md)
  — `Transaction.pack(payer: Account, nodes)` and
  `Transaction.sign(payerId: Address, signer: TransactionSigner, nodes)`.
- Definition of `Account`, `TransactionSigner`, `HieroClient`:
  [`spec/consensus-node-client/client.md`](../../spec/consensus-node-client/client.md).
- Hedera protobuf `QueryHeader.payment`
  (signed CryptoTransfer attached to a paid query):
  [hashgraph/hedera-protobufs](https://github.com/hashgraph/hedera-protobufs).
- V2 reference for sponsored / custom-payer query patterns:
  [Hiero Java SDK `Query.java`](https://github.com/hiero-ledger/hiero-sdk-java/blob/main/sdk/src/main/java/com/hedera/hashgraph/sdk/Query.java)
  — `setPaymentTransaction(Transaction)` /
  `setPaymentTransactionId(TransactionId)`.
