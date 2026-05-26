# Transactions API

This section defines the API for transactions.

## Description

## API Schema

```
namespace consensusnode.transactions
requires ledger, keys, nativeToken, consensusnode.client

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

abstraction Transaction<$$Receipt extends Receipt> {
  
  @@nullable @@immutable maxTransactionFee: nativeToken.NativeToken<ANY, ANY>
  @@nullable @@immutable validDuration: int64
  @@nullable @@immutable memo: string
  @@nullable @@immutable transactionId: ledger.TransactionId
  @@nullable @@immutable maxAttempts: int32
  @@nullable @@immutable maxBackoff: int64
  @@nullable @@immutable minBackoff: int64
  @@nullable @@immutable attemptTimeout: int64
  @@immutable nodeAccountIds: list<common.AccountId>

  // returns a new instance (TODO: this API must be made better)
  Transaction<$$Receipt> sign(account: client.Account)

// returns a new instance (TODO: this API must be made better)
  Transaction<$$Receipt> sign(client: client.HieroClient)
  
  // returns a new instance (TODO: this API must be made better)
  Transaction<$$Receipt> sign(signer: TransactionSigner)
    
  @@async Response<$$Receipt> execute(client: client.HieroClient)

  @@async Response<$$Receipt> signAndExecute(client: client.HieroClient)
  
  bytes toBytes()
}

Response<$$Receipt extends Receipt> {
  @@immutable transactionId: ledger.TransactionId // the id of the transaction

  @@async $$Receipt queryReceipt()          // query for the receipt of the transaction
  @@async Record<$$Receipt> queryRecord()   // query for the record of the transaction
}

abstract Receipt {
  @@immutable transactionId: ledger.TransactionId     // the id of the transaction
  @@immutable status: TransactionStatus        // the status of the transaction
  @@immutable exchangeRate: nativeToken.ExchangeRate     // the exchange rate at the time of the transaction
  @@immutable nextExchangeRate: nativeToken.ExchangeRate // the next exchange rate
}

Record<$$Receipt extends Receipt> {
  @@immutable transactionId: ledger.TransactionId          // the id of the transaction
  @@immutable consensusTimestamp: zonedDateTime      // the consensus time of the transaction
  @@immutable receipt: $$Receipt                     // the typed receipt of the transaction
}


```

## Examples

## Questions & Comments
