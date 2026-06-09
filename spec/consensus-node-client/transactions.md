# Transactions API

This section defines the API for transactions.

## Description

## API Schema

```
namespace consensusnode.transactions
requires {Address, TransactionId} from ledger
requires {NativeToken, ExchangeRate} from nativeToken
requires {Account, HieroClient, NodeSignature, TransactionSigner} from consensusnode.client

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
  @@nullable validDuration: duration
  @@nullable memo: string
  
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> pack(payer: Account, nodes: list<Address>)  
    
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> signWithOperator(client: HieroClient)
  
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> sign(payer: Account, nodes: list<Address>)
  
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> sign(payerId: Address, signer: TransactionSigner, nodes: list<Address>)
  
  @@async Response<$$Receipt> signWithOperatorAndExecute(client: HieroClient)
  
}

abstraction PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> {
  
  @@immutable transactionId: TransactionId
  @@immutable nodeSignatures: list<NodeSignature> 
  @@nullable maxAttempts: int32
  @@nullable maxBackoff: int64
  @@nullable minBackoff: int64
  @@nullable attemptTimeout: int64

  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> sign(account: Account)
  
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> sign(signer: TransactionSigner)

  // Returns the serialized TransactionBody bytes for every target node. Used by out-of-process
  // signing flows (raw HSMs, async signing pipelines, multi-party coordination, audit archival)
  // that cannot be wrapped behind a synchronous TransactionSigner. The returned list has one
  // NodeBody per node in `nodes`.
  list<NodeBody> signableBodies()

  // Attaches externally-produced NodeSignatures to this PackedTransaction and returns a new
  // PackedTransaction containing them. The provided list must contain one signature per node
  // returned by signableBodies() for the same PublicKey; otherwise execute() will fail with
  // INVALID_SIGNATURE on the chosen node.
  // @@throws(unknown-node-error)        if a signature references a node not in `nodes`
  // @@throws(incomplete-signatures-error) if signatures for any target node are missing
  PackedTransaction<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> addSignatures(signatures: list<NodeSignature>)

  @@async Response<$$Receipt> execute(client: HieroClient)
  
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

## Questions & Comments
