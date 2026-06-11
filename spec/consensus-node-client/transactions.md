# Transactions API

This namespace defines the lifecycle of transactions submitted to the consensus node, from a mutable
builder through a wire-ready, signed payload to an executed response.

## Description

A transaction passes through two clearly separated states, modelled as two distinct types:

1. **`Transaction<$$Receipt>`** — a mutable builder. Concrete subtypes (defined under sub-namespaces
   such as `consensusnode.transactions.accounts`) expose service-specific fields; the generic
   transaction-level fields (`maxTransactionFee`, `validDuration`, `memo`) are inherited from this
   abstraction. A `Transaction` is not yet bound to a payer, target nodes, or a `TransactionId`.

2. **`PackedTransaction<$$Receipt, $$Transaction>`** — a frozen, serializable, wire-ready
   transaction. Packing binds the transaction to a payer, a set of target consensus nodes, and a
   freshly generated `TransactionId`. The body is byte-stable from this point on; further
   `NodeSignature` entries may be appended (multi-sig), but no field of the body may change.

The split exists because signing is a function over **exact byte content**: a signature is computed
over the serialized `TransactionBody`, which includes the target node's `nodeAccountID`. Allowing
the body to mutate after signing would silently invalidate every previously collected signature and
break round-trips through serialization. Modelling the two states as two types makes this constraint
visible in the type system rather than relying on documentation or runtime checks.

The naming follows that purpose: `Transaction` is the editable form; `PackedTransaction` is the form
that has been *packed* for the wire (carrying its target nodes, transaction id, and any signatures
collected so far). See ADR-0001 for the full rationale, including why this is preferred over a
single mutable type or a `freeze()` step on `Transaction`.

### Lifecycle

```
  build              pack              sign               submit
   ──▶ Transaction ──▶ PackedTransaction ──▶ PackedTransaction ──▶ Response
                          (0 signatures)        (n signatures)
```

- **build** — construct a concrete `Transaction` subtype and populate its service-specific fields.
- **pack** — bind to a payer `Account`, a list of target nodes, and generate a `TransactionId`.
  Produces a `PackedTransaction` with an empty `nodeSignatures` list. This is the explicit hand-off
  point between the editable and the immutable world.
- **sign** — append one or more `NodeSignature` entries. Several mechanisms are supported
  (see *Signing mechanisms* below).
- **submit** — hand the packed transaction to one of the target consensus nodes and obtain a
  `Response`. The chosen node is an SDK-internal detail; the caller does not steer it. Actual
  ledger execution happens on the network asynchronously — the `Response` is the submission
  acknowledgment, not the execution result. Use `Response.queryReceipt()` /
  `Response.queryRecord()` to read the outcome once consensus is reached. `submit(client)` is
  inherited from `Submittable` (defined in `consensusnode.client`), which also carries the
  shared retry-tuning fields.

A `PackedTransaction` may be serialized via `toBytes()` at any point, shipped to another process or
machine, re-loaded via the static `fromBytes(...)` factory, and have further signatures appended on
the receiving side. The transaction id, the target nodes, and the already-collected signatures all
survive the round-trip.

### Signing mechanisms

The signing surface is intentionally tiered, with one convenience method per common case and one
data-oriented escape hatch for out-of-process workflows:

| Caller has | Method |
|---|---|
| A configured `HieroClient` with operator | `Transaction.signWithOperator(client)` (or `signWithOperatorAndSubmit(client)` for fire-and-forget) |
| A single `Account` that pays *and* signs | `Transaction.sign(payer, nodes)` |
| A separate payer address and a `TransactionSigner` (HSM, hardware wallet, paymaster) | `Transaction.sign(payerId, signer, nodes)` |
| An already-packed transaction that needs another in-process signature (multi-sig) | `PackedTransaction.sign(account)` or `PackedTransaction.sign(signer)` |
| Out-of-process signing (async pipeline, multi-party, audit archival, raw HSM bytes) | `PackedTransaction.signableBodies()` + `PackedTransaction.sign(signatures)` |

Signature ordering is irrelevant — `NodeSignature` entries form a set; the consensus node accepts
them in any order. The packed transaction can also be built first via `Transaction.pack(...)`
without any signature and shipped to one or more signers downstream.

### Why one signing key contributes N signatures

Each `TransactionBody` carries a `nodeAccountID` field naming the consensus node it was prepared
for. The network rejects bodies addressed to a different node (`INVALID_NODE_ACCOUNT`) — a
protocol-level replay protection. When a transaction is packed for N target nodes, N distinct
`TransactionBody` byte sequences are produced, differing only in that field. Consequently each
signing key contributes N signatures, one per `(node, body)` pair. This is encapsulated in the
`NodeSignature` type. `signableBodies()` exposes the matched `NodeBody` payloads for external
signers.

### TransactionId generation

The `TransactionId` (payer + `validStart` timestamp + `scheduled` flag + `nonce`) is generated
during `pack(...)`. The fields are set as follows:

- `payer` — the address of the `Account` passed to `pack(...)`.
- `validStart` — "now minus a small drift offset", set by the SDK.
- `scheduled` — always `false`; the consensus node sets `true` when materializing a
  `ScheduleCreate`.
- `nonce` — always `0`; the consensus node uses non-zero values only for child transactions
  spawned by smart-contract execution.

The `TransactionId` is **never user-settable** through this API. Custom validity windows or
deterministic ids for testing must be modelled by a different mechanism if needed in the future.

## API Schema

```
namespace consensusnode.transactions
requires {Address, TransactionId} from ledger
requires {NativeToken, ExchangeRate} from nativeToken
requires {Account, HieroClient, NodeSignature, Submittable, TransactionSigner} from consensusnode.client

// Defines the status of a transaction. Since we can have custom transaction types based on custom
// services in the consensus node we cannot use an enum here anymore.
abstraction TransactionStatus {
  @@immutable code:int32 // the status code that should be unique based on the consensus node
}

// Defines the status codes that are currently used by services that are part of the consensus node repository
enum BasicTransactionStatus extends TransactionStatus {
    OK
    INVALID_TRANSACTION
    PAYER_ACCOUNT_NOT_FOUND
    ...
    GRPC_WEB_PROXY_NOT_SUPPORTED
}

// A serialized TransactionBody for one target consensus node — the exact bytes an external signer
// must sign. Bytes differ per node because each body carries the target node's nodeAccountID.
type NodeBody {
    @@immutable node: Address
    @@immutable bytes: bytes
}

abstraction Transaction<$$Receipt extends Receipt> {
  
  @@nullable maxTransactionFee: NativeToken<ANY, ANY>
  @@nullable validDuration: seconds
  @@nullable memo: string
  
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> pack(payer: Account, nodes: list<Address>)  
    
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> signWithOperator(client: HieroClient)
  
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> sign(payer: Account, nodes: list<Address>)
  
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> sign(payerId: Address, signer: TransactionSigner, nodes: list<Address>)
  
  @@async Response<$$Receipt> signWithOperatorAndSubmit(client: HieroClient)
  
}

// PackedTransaction is a Submittable that yields a Response when handed to the network.
// Retry-tuning fields (maxAttempts, maxBackoff, minBackoff, attemptTimeout) and the
// submit(client) method are inherited from Submittable.
abstraction PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>>
        extends Submittable<Response<$$Receipt>> {

  @@immutable transactionId: TransactionId
  @@immutable nodeSignatures: list<NodeSignature> 

  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> sign(account: Account)
  
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> sign(signer: TransactionSigner)

  // Returns the serialized TransactionBody bytes for every target node. Used by out-of-process
  // signing flows (raw HSMs, async signing pipelines, multi-party coordination, audit archival)
  // that cannot be wrapped behind a synchronous TransactionSigner. The returned list has one
  // NodeBody per node in `nodes`.
  list<NodeBody> signableBodies()

  // Attaches externally-produced NodeSignatures to this PackedTransaction and returns a new
  // PackedTransaction containing them. The provided list must contain one signature per node
  // returned by signableBodies() for the same PublicKey; otherwise submit() will fail with
  // INVALID_SIGNATURE on the chosen node.
  // @@throws(unknown-node-error)        if a signature references a node not in `nodes`
  // @@throws(incomplete-signatures-error) if signatures for any target node are missing
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> sign(signatures: list<NodeSignature>)

  bytes toBytes()
}

Response<$$Receipt extends Receipt> {
  @@immutable transactionId: TransactionId // the id of the transaction

  @@async $$Receipt queryReceipt()          // query for the receipt of the transaction
  @@async Record<$$Receipt> queryRecord()   // query for the record of the transaction
}

abstraction Receipt {
  @@immutable transactionId: TransactionId     // the id of the transaction
  @@immutable status: TransactionStatus        // the status of the transaction
  @@immutable exchangeRate: ExchangeRate     // the exchange rate at the time of the transaction
  @@immutable nextExchangeRate: ExchangeRate // the next exchange rate
}

Record<$$Receipt extends Receipt> {
  @@immutable transactionId: TransactionId          // the id of the transaction
  @@immutable consensusTimestamp: zonedDateTime      // the consensus time of the transaction
  @@immutable receipt: $$Receipt                     // the typed receipt of the transaction
}

// Factory methods for transaction loading
@@static PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> fromBytes(bytes: bytes)

```

## Examples

The examples below use a hypothetical `AccountCreateTransaction` (a concrete subtype of
`Transaction`, defined under `consensusnode.transactions.accounts`) to illustrate each flow. The
patterns apply to any concrete transaction type.

### 1. Operator does everything

The most common path. The `HieroClient` provides the operator account, the target nodes, and the
network routing.

```
HieroClient client = ...;

Response<AccountCreateReceipt> response = new AccountCreateTransaction()
    .initialBalance(...)
    .key(...)
    .signWithOperatorAndSubmit(client);

AccountCreateReceipt receipt = response.queryReceipt();
Address newAccountId = receipt.accountId;
```

### 2. Distinct payer, single signature

A non-operator `Account` pays for and signs the transaction. The caller supplies the target nodes
explicitly.

```
HieroClient client = ...;
Account payer = ...;            // not necessarily the operator
list<Address> nodes = client.ledger.networkSetting().getConsensusNodes()
                            .map(n -> n.address);

PackedTransaction<...> packed = new AccountCreateTransaction()
    .key(...)
    .sign(payer, nodes);

Response<AccountCreateReceipt> response = packed.submit(client);
```

### 3. Paymaster pattern (sponsor pays, user signs the operation)

The sponsor's `Account` is the payer; the user's `TransactionSigner` (an HSM or hardware wallet)
adds the operation signature. The sponsor signs after the fact on the resulting
`PackedTransaction` — because the protocol requires the payer to sign as well.

```
Account sponsor = ...;
TransactionSigner userSigner = ...;     // wraps HSM / Ledger / etc.
list<Address> nodes = ...;

PackedTransaction<...> packed = new TransferTransaction()
    .addHbarTransfer(...)
    .sign(sponsor.accountId, userSigner, nodes)     // user's operation signature
    .sign(sponsor);                                  // sponsor's payer signature

packed.submit(client);
```

### 4. Multi-sig collected across processes

A treasury account guarded by a `KeyList` is co-signed by three independent custodians. The
`PackedTransaction` is built once, serialized, and shipped to each custodian for signing.

```
// Coordinator builds and packs (no signatures yet):
PackedTransaction<...> packed = new TransferTransaction()
    .addHbarTransfer(treasury, -amount)
    .addHbarTransfer(recipient, +amount)
    .pack(treasuryAccount, nodes);

bytes payload = packed.toBytes();
// payload travels to custodian A (e.g. via signed HTTPS):

// Custodian A:
PackedTransaction<...> received = PackedTransaction.fromBytes(payload);
bytes signedA = received.sign(custodianA).toBytes();
// signedA travels back; coordinator forwards to custodian B, etc.

// Coordinator finalises:
PackedTransaction<...> finalTx = PackedTransaction.fromBytes(signedC);
finalTx.submit(client);
```

### 5. Out-of-process / async signing (server-side pipeline)

An online server packs and persists the transaction, dispatches the signable bodies to a remote
signing service via a queue, and resumes once signatures return — possibly minutes later, possibly
on a different worker.

```
// Web request handler:
PackedTransaction<...> packed = new ContractCallTransaction()
    .contractId(...)
    .functionParameters(...)
    .pack(userAccount, nodes);

db.store(jobId, packed.toBytes());
queue.publish(SigningJob(jobId, userPublicKey, packed.signableBodies()));
return Accepted(jobId);

// Later, on a worker, after signatures arrive via webhook:
PackedTransaction<...> resumed = PackedTransaction.fromBytes(db.load(jobId));
list<NodeSignature> signatures = ...;      // produced by remote signer
PackedTransaction<...> signed = resumed.sign(signatures);
signed.submit(client);
```

## Questions & Comments
