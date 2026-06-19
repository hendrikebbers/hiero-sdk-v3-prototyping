# Queries API

This namespace defines the base abstractions for queries against the consensus node. Concrete
query types (e.g. `AccountBalanceQuery`, `TokenInfoQuery`, `ContractCallQuery`) live in
service-specific sub-namespaces; this file defines only the two shared abstractions and the
cost-discovery shape.

## Description

A query is a read-only call to the consensus node that returns a typed result without changing
ledger state. Two abstractions cover the read surface:

1. **`Query<$$Result>`** — a query that requires no payment. Used for status reads that the
   network exposes for free (account balances, transaction receipts).

2. **`PaidQuery<$$Result>`** — a query that requires payment in the network's native token.
   Extends `Query` and adds a cost-discovery method (`getCost`) plus the `maxQueryPayment`
   ceiling that bounds the auto-discovered price. The query is paid by the client's
   operator; see [ADR-0002](../../docs/adr/0002-defer-paid-query-payer-customization.md)
   for the rationale behind deferring a configurable payer.

Both extend `Submittable<$$Result>` (defined in `consensusnode.client`), the shared execution
abstraction that also backs `PackedTransaction`. From `Submittable` they inherit the retry-tuning
fields (`maxAttempts`, `minBackoff`, `maxBackoff`, `attemptTimeout`) and the `submit(client)`
method that hands the request off to the network. Concrete query types extend `Query` or
`PaidQuery` and add only their own input setters and result type.

The split between `Query` and `PaidQuery` mirrors the protocol distinction the consensus node
makes between free and paid queries, and makes the difference visible at the type level rather
than at runtime. A caller holding a `Query` cannot configure payment that would have no effect;
a caller holding a `PaidQuery` is reminded by the type that payment must be considered.
Promotion from free to paid (should network policy ever change) is a breaking type change
rather than a silent behavioural one.

### Payment model

The consensus node accepts two response modes for a paid query: `COST_ANSWER` returns only the
quoted price; `ANSWER_ONLY` returns the actual data and requires a signed payment transaction
attached to the request. Every `submit(client)` on a `PaidQuery` orchestrates both transparently:

1. The SDK issues a `COST_ANSWER` round-trip to obtain the network's current price for the
   query.
2. If `maxQueryPayment` is set and the quoted price exceeds it, the call fails with
   `max-query-payment-exceeded-error` and no payment is made.
3. Otherwise the SDK builds and signs a payment transaction (a transfer of the quoted amount
   from the client's operator to the chosen consensus node) and issues the `ANSWER_ONLY`
   round-trip carrying that payment.

Callers do not assemble the payment transaction. The auto-discovered price always reflects
the network's current state, so there is no exact-amount escape hatch — callers who need to
bound spend use `maxQueryPayment`; callers who need to know what was actually charged read
`cost` from the `PaidQueryResponse` envelope. The query is paid by the client's operator
in this revision; a configurable payer is deferred and tracked in
[ADR-0002](../../docs/adr/0002-defer-paid-query-payer-customization.md).

`getCost(client)` exposes the `COST_ANSWER` round-trip as a separate operation for cases
where the price must be confirmed before `submit(client)` is called — for example, in UI
confirmation flows or fee-budget pre-computation. The quoted price is not a binding offer;
the network may return a different value on a subsequent call as conditions change. Callers
that want a hard ceiling should set `maxQueryPayment` on the query itself.

### Response envelopes

`submit(client)` does not return the typed payload directly — it returns it wrapped:

- `Query.submit()` → `QueryResponse<$$Result>` with `value` (the typed payload) and
  `answeredBy` (the consensus node that produced the answer, useful for audit and forensics).
- `PaidQuery.submit()` → `PaidQueryResponse<$$Result>`, which extends `QueryResponse` with
  `cost` (the native-token amount actually transferred for this query).

This is structurally parallel to `Response<$$Receipt>` on the transaction side but lighter —
queries are synchronous, so the envelope is purely a metadata wrapper, not a handle to
deferred work. Callers who only need the payload chain `.value` at the end of the call:
`query.submit(client).value`. Callers in billing, audit, or forensics contexts read
`response.cost` and `response.answeredBy` from the envelope.

The free/paid distinction is reflected in the envelope hierarchy: `cost` lives only on
`PaidQueryResponse` because it would be `0` and meaningless on free queries. The same
type-level honesty as the `Query` / `PaidQuery` split itself.

## API Schema

```
namespace consensusnode.queries
requires {AccountId} from ledger
requires {HieroClient, Submittable} from consensusnode.client
requires {NativeToken} from nativeToken

// Envelope around the typed result of a Query. Carries the payload plus minimal
// metadata about how the answer was obtained. Returned by Query.submit().
type QueryResponse<$$T> {
    @@immutable value: $$T
    @@immutable answeredBy: AccountId   // consensus node that produced this answer
}

// Envelope around the typed result of a PaidQuery. Extends QueryResponse with the
// amount actually paid. Returned by PaidQuery.submit().
type PaidQueryResponse<$$T> extends QueryResponse<$$T> {
    @@immutable cost: NativeToken<ANY, ANY>   // amount transferred to the answering node
}

// Base abstraction for any read-only call to the consensus node. Direct subtypes
// represent "free" queries that the network answers without charging the caller
// (e.g. account balance, transaction receipt). Queries that require payment must
// extend PaidQuery instead.
//
// Inherits retry-tuning fields (maxAttempts, maxBackoff, minBackoff, attemptTimeout)
// and submit(client) from Submittable. submit() returns the payload wrapped in a
// QueryResponse envelope.
abstraction Query<$$Result> extends Submittable<QueryResponse<$$Result>> {
}

// A query whose answer requires payment in the network's native token. The price is
// always auto-discovered against the network's current fee schedule on each submit() /
// getCost() — there is no exact-amount lock-in field. Callers who need to bound spend
// use maxQueryPayment; callers who need to know what was actually charged read cost
// from the returned PaidQueryResponse.
//
// Specialises the inherited submit() return type from QueryResponse<$$Result> to
// PaidQueryResponse<$$Result>, so callers can read the actually-paid cost without an
// extra round-trip.
//
// Submit-time errors specific to PaidQuery (in addition to the generic Submittable
// failure modes):
//   - max-query-payment-exceeded-error — auto-discovery returned a price above
//     maxQueryPayment; nothing is sent.
abstraction PaidQuery<$$Result> extends Query<$$Result> {

    // Upper bound on the auto-discovered price. When the quoted cost from the
    // network exceeds this limit, submit() and getCost() fail with
    // max-query-payment-exceeded-error and no payment is made.
    @@nullable maxQueryPayment: NativeToken<ANY, ANY>
    
    // Covariant override: PaidQueryResponse extends QueryResponse.
    @@async PaidQueryResponse<$$Result> submit(client: HieroClient)

    // Issue a COST_ANSWER round-trip and return the network's quoted price for
    // this query without consuming it. The returned value is a snapshot — a
    // subsequent call may return a different price as conditions change.
    // @@throws(max-query-payment-exceeded-error) if maxQueryPayment is set and the quote exceeds it
    @@async NativeToken<ANY, ANY> getCost(client: HieroClient)
}
```

## Examples

The examples below reference hypothetical concrete query types (`AccountBalanceQuery`,
`AccountInfoQuery`) defined in `consensusnode.queries.accounts`. The patterns apply to every
concrete query.

### 1. Free query

```
HieroClient client = ...;

AccountBalance balance = new AccountBalanceQuery()
    .accountId(...)
    .submit(client)
    .value;
```

No payment, no cost discovery, no `maxQueryPayment`. `.value` unwraps the `QueryResponse`
envelope for callers who only need the payload.

### 2. Paid query, default payment

```
AccountInfo info = new AccountInfoQuery()
    .accountId(...)
    .submit(client)
    .value;
```

`maxQueryPayment` is unset. The SDK quotes the price via a cost round-trip and uses the
quoted amount, paid from the client's operator. If the operator has insufficient balance,
the underlying payment transaction fails and the error propagates out of `submit(...)`.

### 3. Paid query with an explicit payment ceiling

```
AccountInfo info = new AccountInfoQuery()
    .accountId(...)
    .maxQueryPayment(NativeToken.of(...))   // refuse to pay more than this
    .submit(client)
    .value;
```

The SDK still quotes via cost discovery, but aborts with `max-query-payment-exceeded-error`
if the quote exceeds the configured ceiling.

### 4. Confirm price up-front

```
PaidQuery<AccountInfo> query = new AccountInfoQuery().accountId(...);

NativeToken<ANY, ANY> cost = query.getCost(client);
// show cost to the user, wait for approval, then:

AccountInfo info = query.submit(client).value;
```

`getCost(client)` performs the cost round-trip only. The subsequent `submit(client)` performs
its own (independent) cost round-trip — the price may have changed between the two calls.
Use `maxQueryPayment` if a hard upper bound is needed.

### 5. Read envelope metadata (audit / billing)

```
PaidQueryResponse<AccountInfo> response = new AccountInfoQuery()
    .accountId(...)
    .submit(client);

AccountInfo info        = response.value;
NativeToken<ANY, ANY> c = response.cost;        // amount actually charged
AccountId answeringNode  = response.answeredBy;  // which node served the query

auditLog.record(info.accountId, c, answeringNode);
```

For free queries, `QueryResponse` carries `value` and `answeredBy`; `cost` is only present on
`PaidQueryResponse` and is therefore unreachable on free-query responses by construction.

## Questions & Comments

- **Envelope adds a `.value` unwrap step.** Every `submit(client)` callsite now ends in
  `.value` (or binds the full envelope when metadata is needed). The friction is real for
  the 95% case that only wants the payload; the wrapper is justified by giving billing /
  audit code a structured place to read `cost` and `answeredBy` without a separate API. If
  this friction proves unacceptable in practice, a future `submitValue(client)` shortcut
  could return the unwrapped payload directly — additive change, no breakage of the
  envelope-returning method.
- **`getCost()` is non-binding.** The returned amount is a quote, not a reservation. The
  network may quote a different price on the subsequent `submit(client)` round-trip. Callers
  who need a hard upper bound must set `maxQueryPayment` rather than relying on a
  `getCost()` value remaining current.
- **No explicit payment override.** PaidQuery deliberately does not expose a
  `queryPayment`-style field that would let the caller pin the exact amount transferred.
  Auto-discovery always runs, so the price reflects the network's current schedule, and
  `maxQueryPayment` covers ceiling semantics. If a real use-case appears (e.g. high-frequency
  signed-fee-cache workflows), an explicit payment field could be added back as an additive
  change — but only with the over/underpayment footguns clearly documented.
- **Free vs. paid is a type-level distinction.** A concrete query that today is free but might
  become paid would migrate from `Query` to `PaidQuery` — a breaking change by design.
- **Configurable payer is deferred.** Today every `PaidQuery` is funded by the client's
  operator. The design space for letting another account pay (paymaster, custodial, audit
  account) is captured in
  [ADR-0002](../../docs/adr/0002-defer-paid-query-payer-customization.md), which compares
  an in-memory `payer: Account` variant against an external-signer `payerId + payerSigner`
  variant and explains why both are deferred. Workaround today: instantiate a separate
  `HieroClient` with the sponsor as operator.
- **Per-query retry semantics** (e.g. `TransactionReceiptQuery` polling on `RECEIPT_NOT_FOUND`
  / `UNKNOWN` until consensus is reached) are concrete-query concerns and will be modelled
  where those queries are defined; the base abstractions deliberately do not encode them.
- **Streaming queries** (e.g. topic message subscriptions) do not fit this request/response
  shape and are modelled separately under `mirrornode.topic` / `enterprise.service.topic` as
  `@@streaming` operations.
- **Should `QueryResponse` carry the ledger origin?** HAPI's `*GetInfoResponse` messages all
  embed a `ledger_id` so a detached payload (mirrored, archived, replayed) can still be
  attributed to its source ledger. The V3 spec drops it from every Info payload (`AccountInfo`
  in [`queries-accounts.md`](queries-accounts.md), `FileInfo` in
  [`queries-files.md`](queries-files.md), and any future Info type) because the caller
  already knows the ledger via their `HieroClient` and duplicating it on every typed payload
  is bloat. The open question is whether to surface it *once* on the envelope — analogous to
  the existing `answeredBy: AccountId` — as
  `@@immutable ledgerId: bytes` (or `LedgerId`, once typed) on `QueryResponse<$$T>`. That
  keeps the metadata available for serialize-then-archive workflows without polluting every
  payload, and applies uniformly to free and paid queries. Defer until a concrete archival /
  audit use-case shows up.
