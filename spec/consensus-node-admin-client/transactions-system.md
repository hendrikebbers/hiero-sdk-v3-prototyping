# System Delete / Undelete Transaction API

## Description

`SystemDelete` and `SystemUndelete` are **privileged moderation transactions** that the
governance council can use to remove or restore a file or contract that violates council
policy (illegal content, malicious bytecode, ...). They are not normal account-level
deletes:

- A *regular* `FileDelete` / `ContractDelete` requires the asset's own `keys` / `adminKey`
  and is irreversible.
- A `SystemDelete` requires the **privileged system-admin or system-delete key**
  (configured at the consensus-node level), bypasses the asset's keys entirely, and can be
  *reversed* by a subsequent `SystemUndelete` until `expirationTime` is reached.

There are two flavours, distinguished by `@@oneOf(fileId, contractId)`:

- **File system-delete** sets the file's "deleted" flag and schedules its permanent
  removal at `expirationTime`. Reads against the file return `FILE_DELETED` during that
  window; `SystemUndelete` clears the flag if invoked before expiration.
- **Contract system-delete** marks the contract as obsolete and prevents new invocations.
  `SystemUndelete` restores access.

`SystemUndelete` does not take an `expirationTime` — it only un-marks. If the file is
already past its `expirationTime` the un-delete fails with `INVALID_FILE_ID`.

### Signing requirements (system delete / undelete)

| Transaction                | Signers required                                                                                      |
|----------------------------|-------------------------------------------------------------------------------------------------------|
| `SystemDeleteTransaction`  | the *payer* **and** the network's privileged *system-admin* (or *system-delete*) key                  |
| `SystemUndeleteTransaction`| the *payer* **and** the network's privileged *system-undelete* key                                    |

The privileged keys are configured by the council out-of-band; SDKs feed them through
`TransactionSigner` exactly like any other externally held key. See
[`transactions.md`](../consensus-node-client/transactions.md) for the HSM-friendly signing
flows.

## API Schema

```
namespace consensusnode.admin.system
requires {Address, ContractId} from ledger
requires {Receipt, Transaction} from consensusnode.transactions

// Marks a file or contract as deleted by privileged council action. Exactly one of fileId /
// contractId must be set. For files, expirationTime is required and marks when the deletion
// becomes permanent (a SystemUndelete before that time reverses it). For contracts,
// expirationTime is ignored.
@@finalType
@@oneOf(fileId, contractId)
SystemDeleteTransaction extends Transaction<SystemDeleteReceipt> {
    @@immutable @@nullable fileId: Address
    @@immutable @@nullable contractId: ContractId
    @@immutable @@nullable expirationTime: zonedDateTime  // when a system-deleted file becomes permanently unrecoverable; required when fileId is set, ignored when contractId is set
}

@@finalType
SystemDeleteReceipt extends Receipt {
}

// Reverses a prior SystemDelete. Exactly one of fileId / contractId must match the prior
// SystemDelete's target. Must be submitted before the file's expirationTime; afterwards
// the un-delete fails with INVALID_FILE_ID.
@@finalType
@@oneOf(fileId, contractId)
SystemUndeleteTransaction extends Transaction<SystemUndeleteReceipt> {
    @@immutable @@nullable fileId: Address
    @@immutable @@nullable contractId: ContractId
}

@@finalType
SystemUndeleteReceipt extends Receipt {
}
```

## Examples

### System-delete a file with a 30-day reversal window

```
HieroClient client = ...;     // council operator; signer wraps the system-admin HSM key
Address offendingFile = ...;

new SystemDeleteTransaction()
    .fileId(offendingFile)
    .expirationTime(zonedDateTime.now().plus(30, DAYS))   // reversible window
    .signWithOperatorAndSubmit(client);
```

### Restore the file inside the reversal window

```
new SystemUndeleteTransaction()
    .fileId(offendingFile)
    .signWithOperatorAndSubmit(client);
```

### System-delete a contract

```
new SystemDeleteTransaction()
    .contractId(maliciousContract)
    .signWithOperatorAndSubmit(client);     // expirationTime not used for contracts
```

## Questions & Comments

- **No first-class system-admin key reference.** Same gap as in
  [`transactions-freeze.md`](transactions-freeze.md): the council-held keys that authorise
  system delete / undelete are not modelled on the transaction; they reach the SDK through
  `TransactionSigner`. A future `NetworkAdminKey` type could surface the distinction
  between *system-admin*, *system-delete*, and *system-undelete* keys, but is not in scope
  today.

- **Splitting `SystemDelete` into `SystemDeleteFileTransaction` /
  `SystemDeleteContractTransaction`** — same trade-off as the `freezeType` enum question:
  the `@@oneOf` mirrors HAPI but loses some compile-time safety (`expirationTime` is only
  valid in the file branch, yet sits next to `contractId` in the type). Two concrete
  subtypes would remove the cross-field dependency. Decide before locking the API.

- **No status code yet for "expired window".** A `SystemUndelete` on a file whose
  reversal window has elapsed surfaces as `INVALID_FILE_ID`, which is indistinguishable
  from "file never existed". Whether the SDK should map this case to a dedicated
  `system-undelete-window-expired-error` is open.
