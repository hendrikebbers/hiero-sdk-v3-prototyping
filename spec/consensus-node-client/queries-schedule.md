# Schedule Queries API

This namespace defines the read-side counterpart to the schedule transactions in
[`transactions-schedule.md`](transactions-schedule.md). One paid query is exposed:
`ScheduleInfoQuery` returns the current state of a stored schedule.

## Description

- **`ScheduleInfoQuery`** extends `PaidQuery<ScheduleInfo>` — returns a metadata snapshot of a
  schedule: who created it, who pays the inner transaction's fee, the captured inner transaction,
  the keys collected so far, and whether the schedule is still pending, has executed, or was
  deleted.

The query extends `PaidQuery`, so it inherits the `maxQueryPayment` knob, the `getCost(client)`
cost-discovery method, and the `PaidQueryResponse<...>` envelope. See [`queries.md`](queries.md)
for the full payment / envelope semantics.

### Pending / executed / deleted

A schedule is in exactly one of three states, captured by `@@oneOrNoneOf(executionTime,
deletionTime)`:

- **pending** — neither timestamp is set; the schedule is still collecting signatures (or, for an
  HIP-423 long-term schedule, waiting for `expirationTime`).
- **executed** — `executionTime` is set to the consensus time at which the inner transaction ran
  once its required signatures were collected.
- **deleted** — `deletionTime` is set to the consensus time at which a `ScheduleDelete` removed the
  schedule before it could execute.

Unlike `TopicInfo` (where a removed topic disappears from the consensus node), a deleted or executed
schedule remains queryable until it is reaped, so the state is modelled as data on the snapshot
rather than as a query failure.

### The inner transaction

The captured inner transaction is exposed as a `Transaction<ANY>` — the same builder type
`ScheduleCreate` accepts (see [`transactions-schedule.md`](transactions-schedule.md)). It is a
read-only reconstruction of the stored body; it is never packed or signed. To read the *outcome* of
an executed schedule, use `scheduledTransactionId` with the `Transaction.getResponse(...)` factory
in [`transactions.md`](transactions.md), exactly as shown in the schedule examples.

### Signers

`signers` lists the individual public keys whose signatures the network has already credited toward
the inner transaction's required key set. It is `list<PublicKey>` (concrete keys that have signed),
not `list<Authority>` — a signer is a single key that produced a signature, whereas an `Authority`
is an authorization *requirement* (see [`authority.md`](../base/authority.md)). `adminKey`, by
contrast, is an `Authority`: it is the requirement that must be met to delete the schedule.

## API Schema

```
namespace consensusnode.queries.schedule
requires {Address, AccountId, TransactionId} from ledger
requires {PublicKey} from keys
requires {Authority} from authority
requires {PaidQuery} from consensusnode.queries
requires {Transaction} from consensusnode.transactions

// Full metadata snapshot of a schedule. Returned by ScheduleInfoQuery.
@@oneOrNoneOf(executionTime, deletionTime)
type ScheduleInfo {
    @@immutable scheduleId: Address                        // the id of the schedule
    @@immutable creatorAccountId: AccountId                // account that submitted the ScheduleCreate
    @@immutable payerAccountId: AccountId                  // pays the inner transaction's fee when it executes
    @@immutable scheduledTransaction: Transaction<ANY>     // read-only reconstruction of the captured inner transaction
    @@immutable scheduledTransactionId: TransactionId      // the id the inner transaction carries when it executes (scheduled flag set)
    @@immutable @@default([]) signers: list<PublicKey>     // public keys whose signatures have been credited so far
    @@immutable @@nullable adminKey: Authority             // may delete the schedule; unset → schedule is immutable
    @@immutable @@nullable scheduleMemo: string            // free-form memo on the schedule entity
    @@immutable expirationTime: zonedDateTime              // when the schedule expires (HIP-423 long-term, or the default expiry)
    @@immutable @@default(false) waitForExpiry: bool       // HIP-423 long-term: execute only at expirationTime even if signatures are collected earlier
    @@immutable @@nullable executionTime: zonedDateTime    // set once the inner transaction has executed
    @@immutable @@nullable deletionTime: zonedDateTime     // set once the schedule has been deleted before execution
}

// Paid query for the current state of a stored schedule.
@@finalType
ScheduleInfoQuery extends PaidQuery<ScheduleInfo> {
    @@immutable scheduleId: Address
}
```

## Examples

### Read a schedule's state (paid)

```
HieroClient client = ...;

ScheduleInfo info = new ScheduleInfoQuery()
    .scheduleId(scheduleId)
    .submit(client)
    .value;

list<PublicKey> signed = info.signers;            // keys credited so far
Authority       admin  = info.adminKey;            // null if the schedule is immutable
```

### Branch on pending / executed / deleted

```
ScheduleInfo info = new ScheduleInfoQuery()
    .scheduleId(scheduleId)
    .submit(client)
    .value;

if (info.executionTime != null) {
    // executed — read the inner transaction's outcome with its type token
    Response<TransferReceipt> innerResponse =
        Transaction.getResponse(info.scheduledTransactionId, TransferTransaction, client);
    TransferReceipt innerReceipt = innerResponse.queryReceipt();
} else if (info.deletionTime != null) {
    // deleted before execution
} else {
    // still pending — collect more signatures via ScheduleSign
}
```

## Questions & Comments

- **`signers` is `list<PublicKey>`, not `list<Authority>`.** A credited signer is always a single
  concrete public key that produced a valid signature; the recursive `Authority` sum type
  (single key / contract / m-of-n) models an authorization *requirement*, not a signature that has
  been collected. HAPI returns `signers` as a `KeyList`, but every entry is a simple key, so the
  flatter `list<PublicKey>` is both honest and lighter. `adminKey` stays an `Authority` because it
  *is* a requirement (matching `ScheduleCreateTransaction.adminKey` in
  [`transactions-schedule.md`](transactions-schedule.md)).

- **The inner transaction is `Transaction<ANY>`.** Same modelling choice as
  `ScheduleCreateTransaction.scheduledTransaction`: the captured body has no compile-time receipt
  type. Reading its execution outcome therefore re-supplies the inner transaction's type token to
  `Transaction.getResponse(...)` (see [`transactions.md`](transactions.md) and the schedule
  examples). The snapshot exposes the *captured body*; it is never packed or signed.

- **No `deleted` boolean — state is the `@@oneOrNoneOf` timestamp pair.** Rather than a flag plus a
  timestamp, the executed/deleted/pending state is read directly from which (if either) of
  `executionTime` / `deletionTime` is set, matching HAPI's `oneof { execution_time, deletion_time }`.
  A pending schedule has neither.

- **No `ledgerId` on `ScheduleInfo`.** Matches `TopicInfo` / `AccountInfo` / `FileInfo` — the caller
  already knows the ledger via their `HieroClient`, and ledger-origin metadata belongs on the
  response envelope, not on every payload. See the open envelope-level question in
  [`queries.md`](queries.md) *Questions & Comments*.

- **`customFeeLimits` (HIP-991) is absent**, consistent with its absence on
  `ScheduleCreateTransaction` — it depends on the write-side custom-fee model not yet specified in
  V3 (§3.3 in [`missing-features.md`](../../missing-features.md)). Additive once that lands.
