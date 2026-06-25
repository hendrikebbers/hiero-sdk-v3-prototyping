# ADR-0005: Model the schedule service on the existing transaction model, without schedule-specific types

**Status:** Accepted
**Date:** 2026-06-24

## Context

The schedule service ([HIP-423 long-term scheduled transactions](https://hips.hedera.com/HIP/hip-423.html))
lets a caller store a transaction on the ledger that executes later, once the
signatures its own authorization requires have been collected. V3 must expose
`ScheduleCreate`, `ScheduleSign`, and `ScheduleDelete`
([`missing-features.md`](../../missing-features.md) §1.6).

The challenge is that a scheduled transaction is structurally unlike every other
transaction the V3 API already models:

### The inner transaction does not fit the pack/sign lifecycle

V3 models a transaction as a two-state lifecycle — an editable
`Transaction<$$Receipt>` builder that is `pack(...)`-ed into a byte-stable
`PackedTransaction` and then signed (ADR-0001,
[`spec/consensus-node-client/transactions.md`](../../spec/consensus-node-client/transactions.md)).
A signature is computed over the serialized `TransactionBody`, which carries the
target node's `nodeAccountID`; this is why packing binds a transaction to its
target nodes and a `TransactionId` before any signature exists
([`transactions.md`](../../spec/consensus-node-client/transactions.md):42-83).

`ScheduleCreate`, by contrast, captures only the **body** of an inner
transaction. The inner transaction has no `nodeAccountID`, is never submitted in
its own right, and must never be packed or signed by the caller — the consensus
node materializes it. So the central lifecycle operation (`pack`) is meaningless
for the inner transaction.

### The set of schedulable transactions is open

HAPI restricts which transactions may be scheduled, but the V3 consensus-node
client is deliberately extensible: the SDK supports custom services and custom
transaction types via an SPI keyed by transaction type
([`spec/consensus-node-client/transactions-spi.md`](../../spec/consensus-node-client/transactions-spi.md)).
This is the same openness that forced `TransactionStatus` to be an `abstraction`
with an `int32` code rather than a closed enum
([`transactions.md`](../../spec/consensus-node-client/transactions.md):109-121).
The SDK cannot know whether a future custom transaction type is schedulable.

### Schedule signatures look like a second signature concept — but are not

A scheduled transaction accumulates signatures on the ledger over time. The
question is whether this needs a dedicated "schedule signature" payload distinct
from the per-node `NodeSignature` used everywhere else. The V2 SDKs answer this:
signatures are added by **signing the `ScheduleSign` (or `ScheduleCreate`)
transaction normally** with the required keys — "Required parties can use a
`ScheduleSignTransaction` to add their signatures to the scheduled transaction",
and "the transaction will execute immediately after sufficient signatures are
received" unless `wait_for_expiry` defers it
([Hedera scheduled-transaction docs](https://docs.hedera.com/hedera/core-concepts/scheduled-transaction)).
The network performs key-presence accounting against the inner transaction's
required key set; there is no separate signature object the caller constructs.

This decision was driven out in a `/grill-me` session covering the inner-type
model, schedulability enforcement, the signing model, the field set, custom-fee
limits, deletion/receipts, and how the eventual execution result is read.

## Decision

We will model the schedule service **entirely on top of the existing transaction
model**, adding no schedule-specific type machinery
([`spec/consensus-node-client/transactions-schedule.md`](../../spec/consensus-node-client/transactions-schedule.md)).
Concretely:

1. **The inner transaction is captured as a plain `Transaction<ANY>`** — the same
   builder type used everywhere else. `ScheduleCreate` captures only its body; the
   inherited `pack()`/`sign()` methods are simply never called on the inner
   instance. Calling them is harmless (the builder returns an unrelated object and
   is not mutated), so no new type is needed to forbid them.

2. **The SDK performs no schedulability check.** A non-schedulable inner
   transaction is rejected by the consensus node as a `TransactionStatus` on the
   receipt, not by a client-side error or `@@throws`. The SDK cannot decide
   schedulability for SPI-provided custom transaction types, so it does not try.

3. **Schedule signatures reuse the existing `pack()`/`sign()` multi-signature
   flow.** `ScheduleSign` carries only `scheduleId`; contributing a signature is
   an ordinary `sign(account)` on the outer transaction. A schedule signature *is*
   a normal `NodeSignature` over the outer transaction; the network does the
   key-presence accounting. No `ScheduleSignature` payload is introduced.

We **reject** the alternatives below.

## Alternatives considered

### A. A dedicated `SchedulableTransaction` type without `pack()`/`sign()`

Rejected. To offer a schedulable type that structurally lacks the irrelevant
`pack()`/`sign()` surface, the service-specific fields (e.g. the
`hbarTransfers`/`tokenTransfers`/`nftTransfers` of a transfer,
[`transactions-accounts.md`](../../spec/consensus-node-client/transactions-accounts.md):147-152)
would have to live in a separate type per operation — duplicating the entire
transaction hierarchy, or restructuring every transaction to draw its fields from
a shared `…Body`. The cost is disproportionate to the benefit: the only thing the
separate type buys is hiding two methods whose misuse cannot break anything.

### B. A marker abstraction `SchedulableTransaction` to enforce schedulability in the type system

Rejected. A closed marker hierarchy that only schedulable types extend collides
with the open SPI service model
([`transactions-spi.md`](../../spec/consensus-node-client/transactions-spi.md)):
custom services bring their own transaction types, and the SDK cannot know whether
they are schedulable. Type-level enforcement would either exclude legitimate
custom transactions or give a false guarantee. The network is the only component
that knows the true schedulable set.

### C. A separate `ScheduleSignature` payload on `ScheduleSign`/`ScheduleCreate`

Rejected. V2 semantics show schedule signatures are ordinary signatures on the
outer transaction, applied by the network via key-presence accounting
([Hedera scheduled-transaction docs](https://docs.hedera.com/hedera/core-concepts/scheduled-transaction)).
A bespoke payload would invent a second signing concept parallel to
`NodeSignature` for no protocol reason, and would clash with the existing
out-of-process/HSM signing surface (`signableBodies()`, `sign(list<NodeSignature>)`).

## Consequences

### Positive

- The schedule service adds **three transaction types and three receipts** and no
  new core concept. It rides the existing build → pack → sign → submit lifecycle,
  including the offline/HSM/multi-party signing flows, unchanged.
- Multi-party schedule signing is literally the existing multi-signature flow;
  there is nothing new to learn or document for it.
- The open-world stance is consistent with the rest of the consensus-node client
  (`TransactionStatus` abstraction, the SPI), so custom services that add
  schedulable transactions work without any change to the schedule types.

### Negative

- The inner `Transaction<ANY>` exposes `pack()`/`sign()` that are meaningless in
  the schedule context. The constraint "do not pack the inner transaction" is
  documented, not type-enforced — a deliberate trade against hierarchy
  duplication (Alternative A).
- Passing a non-schedulable transaction is only caught at the network, surfacing
  later (on the receipt) than a client-side check would. Accepted as the price of
  the open SPI model.
- The inner transaction is `Transaction<ANY>`, so `scheduledTransactionId` carries
  no compile-time link to the inner receipt type. Reading the executed inner
  transaction's outcome in a typed way required adding a general
  `@@static Response<$$Receipt> getResponse(transactionId, transactionType, client)`
  factory to [`transactions.md`](../../spec/consensus-node-client/transactions.md);
  the caller must re-supply the inner transaction's type token.

### Follow-ups

- `customFeeLimits` ([HIP-991](https://hips.hedera.com/hip/hip-991)) is
  intentionally deferred — its payload depends on the write-side custom-fee model
  that is not yet specified
  ([`missing-features.md`](../../missing-features.md) §3.3), the same blocker as
  `TokenFeeScheduleUpdate`. Revisit when the write-side `CustomFee` hierarchy lands.
- `ScheduleInfoQuery` (read-side schedule state) remains unspecified
  ([`missing-features.md`](../../missing-features.md) §1.8).
- Revisit this decision if a future protocol change makes schedule signatures
  diverge from ordinary transaction signatures, or introduces a schedulability
  signal the SDK could check locally.

---

**References:**

- Spec produced by this decision: [`spec/consensus-node-client/transactions-schedule.md`](../../spec/consensus-node-client/transactions-schedule.md)
- Transaction lifecycle, `NodeSignature` fan-out, `TransactionStatus` abstraction, `getResponse` factory: [`spec/consensus-node-client/transactions.md`](../../spec/consensus-node-client/transactions.md) (pack/sign 42-83; TransactionStatus 109-121; scheduled flag 87-94)
- Open SPI service model keyed by transaction type: [`spec/consensus-node-client/transactions-spi.md`](../../spec/consensus-node-client/transactions-spi.md)
- Duplication cost reference (per-operation service fields): [`spec/consensus-node-client/transactions-accounts.md`](../../spec/consensus-node-client/transactions-accounts.md):147-152
- Two-type lifecycle rationale: [ADR-0001](0001-transaction-and-packed-transaction-split.md)
- Feature tracking: [`missing-features.md`](../../missing-features.md) §1.6, §1.8, §1.9, §3.3
- HIP-423 Long Term Scheduled Transactions (`wait_for_expiry`, `expiration_time`, 2-month window): [hips.hedera.com/HIP/hip-423.html](https://hips.hedera.com/HIP/hip-423.html) (accessed 2026-06-24)
- HIP-991 max_custom_fees / `CustomFeeLimit`: [hips.hedera.com/hip/hip-991](https://hips.hedera.com/hip/hip-991) (accessed 2026-06-24)
- V2 schedule signing & deletion semantics (sign normally; execute on sufficient signatures; admin-key-gated deletion): [Hedera scheduled-transaction docs](https://docs.hedera.com/hedera/core-concepts/scheduled-transaction) (accessed 2026-06-24)
