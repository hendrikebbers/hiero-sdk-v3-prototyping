# Network Freeze Transaction API

## Description

`FreezeTransaction` is the **only mechanism** by which the consensus network can be paused
or upgraded. It is authorised by the network's *freeze key* (configured at the consensus
node out-of-band — not the operator's normal account key), which is held by the governance
council and rotates independently of any application's keys. Submitting a
`FreezeTransaction` from a non-privileged payer fails with `AUTHORIZATION_FAILED`; this
transaction has no use case for end-user applications and is the reason the
`consensus-node-admin-client` module exists.

The transaction encodes one of five distinct operations selected by `freezeType`:

- **`FREEZE_ONLY`** — pauses consensus at `startTime`. The network remains frozen until
  manually resumed by the council. No upgrade payload is required.
- **`PREPARE_UPGRADE`** — does **not** freeze the network. Stages the upgrade payload
  identified by `updateFile` (a system file containing the upgrade archive) for a later
  `FREEZE_UPGRADE`. The network keeps running, `startTime` is ignored, and `updateFile` /
  `fileHash` are required. The consensus node rejects the transaction if the hash of the
  file's current contents does not match `fileHash`.
- **`FREEZE_UPGRADE`** — at `startTime`, freezes the network and applies the previously
  staged upgrade. Must be preceded by a matching `PREPARE_UPGRADE`, otherwise the
  consensus node rejects with `NO_UPGRADE_HAS_BEEN_PREPARED`.
- **`FREEZE_ABORT`** — cancels a pending `FREEZE_ONLY` or `FREEZE_UPGRADE` whose
  `startTime` has not yet arrived. `startTime` / `updateFile` / `fileHash` are ignored.
- **`TELEMETRY_UPGRADE`** — applies an upgrade to the telemetry / observability subsystem
  only. The consensus path stays live (no downtime). Requires `updateFile` and `fileHash`
  like `PREPARE_UPGRADE` / `FREEZE_UPGRADE`.

Typical upgrade flow:

```
  PREPARE_UPGRADE ──▶ (network stays live, nodes pre-stage the upgrade)
                  ──▶ FREEZE_UPGRADE @ startTime
                  ──▶ (network freezes, applies upgrade, council restarts)
```

### Signing requirements (freeze)

| Transaction         | Signers required                                                                          |
|---------------------|-------------------------------------------------------------------------------------------|
| `FreezeTransaction` | the *payer* **and** the network's privileged *freeze key* (configured at the consensus node) |

The freeze key is not modelled by this spec — it lives in the network's address-book
configuration and is signed by the council's HSM. The
`PackedTransaction.sign(signer)` / `PackedTransaction.sign(signatures)` flows from
[`transactions.md`](../consensus-node-client/transactions.md) are the right entry points
for HSM-backed council signing.

## API Schema

```
namespace consensusnode.admin.freeze
requires {Address} from ledger
requires {Receipt, Transaction} from consensusnode.transactions

// Selects which of the five operations to perform. The per-value field requirements
// (described in the file header) are enforced server-side; the consensus node rejects
// mismatched payloads with INVALID_FREEZE_TRANSACTION_BODY.
enum FreezeType {
    FREEZE_ONLY
    PREPARE_UPGRADE
    FREEZE_UPGRADE
    FREEZE_ABORT
    TELEMETRY_UPGRADE
}

@@finalType
FreezeTransaction extends Transaction<FreezeReceipt> {
    @@immutable freezeType: FreezeType
    @@immutable @@nullable startTime: zonedDateTime          // wall-clock time at which the network freezes; required for FREEZE_ONLY, FREEZE_UPGRADE, TELEMETRY_UPGRADE; ignored for PREPARE_UPGRADE, FREEZE_ABORT
    @@immutable @@nullable updateFile: Address               // id of a file holding the upgrade payload; required for PREPARE_UPGRADE, FREEZE_UPGRADE, TELEMETRY_UPGRADE
    @@immutable @@nullable fileHash: bytes                   // SHA-384 of the upgrade file's contents; required whenever updateFile is set; the consensus node rejects with FREEZE_UPDATE_FILE_HASH_DOES_NOT_MATCH on mismatch
}

@@finalType
FreezeReceipt extends Receipt {
}
```

## Examples

### Prepare and execute a network upgrade

```
HieroClient client = ...;           // operator signer wraps the council freeze key (HSM)

Address upgradeFile = ...;           // system file (e.g. 0.0.150) containing the upgrade archive
bytes   upgradeHash = ...;           // SHA-384 of the file contents

// Step 1 — stage the upgrade; the network keeps running.
new FreezeTransaction()
    .freezeType(FreezeType.PREPARE_UPGRADE)
    .updateFile(upgradeFile)
    .fileHash(upgradeHash)
    .signWithOperatorAndSubmit(client);

// Step 2 — freeze and apply at startTime.
new FreezeTransaction()
    .freezeType(FreezeType.FREEZE_UPGRADE)
    .startTime(...)
    .updateFile(upgradeFile)
    .fileHash(upgradeHash)
    .signWithOperatorAndSubmit(client);
```

### Schedule a pause and then abort it

```
// Schedule a pause:
new FreezeTransaction()
    .freezeType(FreezeType.FREEZE_ONLY)
    .startTime(...)
    .signWithOperatorAndSubmit(client);

// Decide it was too aggressive — abort while startTime is still in the future:
new FreezeTransaction()
    .freezeType(FreezeType.FREEZE_ABORT)
    .signWithOperatorAndSubmit(client);
```

## Questions & Comments

- **Per-`freezeType` field requirements are validated server-side, not in the type
  system.** A more type-safe alternative would be five concrete subtypes
  (`FreezeOnlyTransaction`, `PrepareUpgradeTransaction`, `FreezeUpgradeTransaction`,
  `FreezeAbortTransaction`, `TelemetryUpgradeTransaction`). That would also remove the
  three `@@nullable` fields whose validity depends on `freezeType`. The trade-off is that
  the current single-type / single-enum surface matches HAPI 1-to-1, which simplifies
  protobuf round-tripping for tooling. Decide before locking the API.

- **The freeze key is not modelled on the transaction.** The privileged signer is
  configured at the consensus-node level (council-controlled), not declared per
  transaction.

- **No streaming progress signal for `FREEZE_UPGRADE`.** The receipt confirms only that
  the consensus node accepted the freeze schedule, not that the upgrade has been applied.
  Observing the actual restart still requires mirror-node / health-check polling outside
  this transaction's surface. Whether the admin client should expose a typed
  "upgrade-applied" event is open.
