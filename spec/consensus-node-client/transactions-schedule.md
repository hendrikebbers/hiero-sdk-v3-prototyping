# Schedule Transactions API


## Description

This namespace groups the schedule-service transactions: `ScheduleCreate` (store a transaction
on the ledger to be executed later, once its required signatures are collected), `ScheduleSign`
(contribute a signature toward a stored transaction), and `ScheduleDelete` (remove a stored
transaction before it executes).

A *scheduled transaction* is an ordinary transaction whose execution is deferred. `ScheduleCreate`
captures an **inner transaction** and persists its body on the network under a new `scheduleId`.
The inner transaction executes automatically once the network has collected every signature its
own authorization requires (or, for HIP-755 long-term schedules, when the schedule expires — see
`waitForExpiry`).

### The inner transaction

The inner transaction is modelled as a plain `Transaction<ANY>` — the *same* builder type used
everywhere else. `ScheduleCreate` only captures its body; it is **never packed or signed itself**.
The `pack()` / `sign()` methods inherited from `Transaction` are simply not used on the inner
instance — calling them produces an unrelated `PackedTransaction` that the schedule flow ignores;
the inner builder is left untouched.

There is **no SDK-side check** that the inner transaction is actually schedulable. HAPI restricts
the schedulable set (e.g. a `ScheduleCreate`, `ScheduleSign`, `ScheduleDelete`, or `freeze` cannot
be scheduled), but because the consensus node is service-oriented and supports *custom* services
and transaction types (see `consensusnode.transactions.spi`), the SDK cannot know whether a future
custom transaction type is schedulable. A non-schedulable inner transaction is therefore rejected
by the network as a `TransactionStatus` on the receipt, not by a client-side error.

### Signing model

Signatures that authorize the inner transaction are **ordinary signatures on the outer schedule
transaction** — there is no separate "schedule signature" payload:

- Any signature placed on a `ScheduleCreate` (beyond the payer) whose public key belongs to the
  inner transaction's required key set is credited to the schedule immediately on creation.
- `ScheduleSign` carries only the `scheduleId`. Each required key signs the `ScheduleSign`
  transaction through the normal `pack()` / `sign(...)` multi-signature flow; the network extracts
  the public keys from the signature set, verifies them against the `ScheduleSign` body, and
  credits those that match the inner transaction's required keys (key-presence accounting).

This means schedule signing needs **no new concept** on top of the existing `Transaction` →
`PackedTransaction` lifecycle: a schedule signature *is* a normal `NodeSignature` over the outer
transaction. The per-node body fan-out (one body per target node, one submitted, the rest spare)
is exactly the standard model from [`transactions.md`](transactions.md).

### Lifecycle and deletion

`ScheduleDelete` removes a stored schedule **before** it executes; it requires a signature from the
schedule's `adminKey`. A schedule created without an `adminKey` is immutable and cannot be deleted
— it can only execute or expire. Deleting an already-executed (or already-deleted) schedule is
rejected by the network as a `TransactionStatus`.

The `scheduled` flag of the inner transaction's `TransactionId` is set by the consensus node when
it materializes the inner transaction (see *TransactionId generation* in
[`transactions.md`](transactions.md)); it is never set by the SDK.

## API Schema

```
namespace consensusnode.transactions.schedule
requires {Address, AccountId, TransactionId} from ledger
requires {Authority} from authority
requires {Receipt, Transaction} from consensusnode.transactions

// Stores an inner transaction on the ledger for deferred execution. The inner transaction is a
// regular Transaction builder; only its body is captured — it is never packed or signed itself.
// Additional signatures placed on this ScheduleCreate (beyond the payer) that match the inner
// transaction's required keys are credited to the schedule on creation.
@@finalType
ScheduleCreateTransaction extends Transaction<ScheduleCreateReceipt> {
    @@immutable scheduledTransaction: Transaction<ANY>     // the inner transaction to execute later; only its body is captured (never packed/signed)
    @@immutable @@nullable adminKey: Authority             // may delete the schedule before execution; unset → schedule is immutable and cannot be deleted
    @@immutable @@nullable payerAccountId: AccountId       // pays the fee of the inner transaction when it executes; null → the payer of this ScheduleCreate pays it
    @@immutable @@nullable scheduleMemo: string            // free-form memo on the schedule entity
    @@immutable @@nullable expirationTime: zonedDateTime   // HIP-755 long-term: when the schedule expires
    @@immutable @@default(false) waitForExpiry: bool       // HIP-755 long-term: when true, execute only at expirationTime even if the required signatures are collected earlier
}

@@finalType
ScheduleCreateReceipt extends Receipt {
    @@immutable scheduleId: Address                        // the id of the newly created schedule
    @@immutable scheduledTransactionId: TransactionId      // the id the inner transaction carries when it executes (the scheduled flag is set on it)
}

// Contributes one or more signatures toward a stored schedule. The transaction carries only the
// scheduleId; the actual authorization comes from signing this transaction with the required keys
// through the normal multi-signature flow (see transactions.md).
@@finalType
ScheduleSignTransaction extends Transaction<ScheduleSignReceipt> {
    @@immutable scheduleId: Address                        // the schedule to add signatures to
}

@@finalType
ScheduleSignReceipt extends Receipt {
    @@immutable scheduledTransactionId: TransactionId      // the id of the inner transaction (populated whether or not this signature triggered execution)
}

// Deletes a stored schedule before it executes. Requires a signature from the schedule's adminKey;
// a schedule created without an adminKey cannot be deleted.
@@finalType
ScheduleDeleteTransaction extends Transaction<ScheduleDeleteReceipt> {
    @@immutable scheduleId: Address                        // the schedule to delete; only valid before execution
}

@@finalType
ScheduleDeleteReceipt extends Receipt {
}
```

## Examples

### Schedule a transfer and contribute the operator's signature

The operator stores a transfer that moves value out of Alice's account. The operator signs the
`ScheduleCreate` as payer; because Alice's signature is still missing, the inner transfer does not
execute yet. The receipt yields the `scheduleId` (to collect more signatures) and the
`scheduledTransactionId` (the id the transfer will carry once it executes).

```
HieroClient client = ...;     // operator pays for the ScheduleCreate
AccountId alice = ...;
AccountId bob = ...;

TransferTransaction inner = new TransferTransaction()
    .hbarTransfers([
        new HbarTransfer(alice, NativeToken.of(-10, HBAR_UNIT)),
        new HbarTransfer(bob,   NativeToken.of(+10, HBAR_UNIT)),
    ]);

Response<ScheduleCreateReceipt> response = new ScheduleCreateTransaction()
    .scheduledTransaction(inner)
    .adminKey(operatorAuthority)        // optional: lets the operator delete it later
    .signWithOperatorAndSubmit(client);

ScheduleCreateReceipt receipt = response.queryReceipt();
Address scheduleId = receipt.scheduleId;
TransactionId scheduledTransactionId = receipt.scheduledTransactionId;
```

### Add a missing signature via ScheduleSign

Alice contributes her signature to the pending schedule. `ScheduleSign` carries only the
`scheduleId`; Alice's authorization is the ordinary signature on the `ScheduleSign` transaction.
The operator pays the fee, Alice co-signs — the same multi-signature flow as any other transaction
(see [`transactions.md`](transactions.md)). Once Alice's key satisfies the inner transfer's
requirements, the network executes it automatically.

```
HieroClient client = ...;     // operator pays the ScheduleSign fee
Account alice = ...;          // the account whose debit the inner transfer needs

PackedTransaction<...> packed = new ScheduleSignTransaction()
    .scheduleId(scheduleId)
    .signWithOperator(client)   // operator signs as payer
    .sign(alice);               // Alice's signature is credited to the schedule

Response<ScheduleSignReceipt> response = packed.submit(client);
TransactionId executed = response.queryReceipt().scheduledTransactionId;
```

### Delete a schedule before it executes

The holder of the schedule's `adminKey` removes it before the remaining signatures arrive.

```
new ScheduleDeleteTransaction()
    .scheduleId(scheduleId)
    .signWithOperatorAndSubmit(client);   // operator holds the adminKey
```

## Questions & Comments

- **`customFeeLimits` (HIP-991) is intentionally absent.** It bounds the custom fees the payer is
  willing to pay for the inner transaction, and its payload depends on the **write-side custom-fee
  model, which is not yet specified in V3** (§3.3 in [`missing-features.md`](../../missing-features.md)).
  This is the same reason `TokenFeeScheduleUpdate` is deferred (see
  [`transactions-tokens-management.md`](transactions-tokens-management.md)). It will be added once
  the write-side `CustomFee` hierarchy exists.

- **Typed reading of the scheduled execution's outcome is out of scope.** The inner transaction is
  `Transaction<ANY>`, so `scheduledTransactionId` is a plain `TransactionId` with no compile-time
  link to the inner receipt type. Reading the inner transaction's receipt/record once it executes
  (possibly much later for long-term schedules) needs a *query-by-id* capability that the current
  model lacks — typed receipts are only reachable through the `Response<$$Receipt>` returned by
  `submit()`. A general `Response<$$Receipt> response(transactionId, receiptType)` on the client /
  [`transactions.md`](transactions.md) (using the SPI `TransactionSupport` to map the type) is the
  prerequisite; until it exists, callers read the scheduled execution untyped via the Mirror Node.
  Tracked in [`missing-features.md`](../../missing-features.md) §1.9.

- **`ScheduleInfoQuery` is not specified here** — read-side schedule state belongs with the other
  consensus-node queries (still missing, see [`missing-features.md`](../../missing-features.md)
  §1.8).
