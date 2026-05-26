# Transactions SPI API

This section defines the SPI API for transactions.

## Description

Since the consensus node is designed in a service-oriented manner, transactions are handled by a separate service.
The consensus node supports adding custom services next to the services that are part of the consensus node repository.
Since new and custom services can provide new transaction types, the SDKs must be able to handle these new transaction types.
The transactions SPI API defines the interface that must be implemented by the custom service that provides new transaction types.

In the `TransactionBuilder`/`Transaction` model, the domain-specific type is the builder (e.g. `AccountCreateTransactionBuilder`).
`Transaction` itself is universal and has no domain-specific subclasses. Therefore, the SPI converts between protobuf types and the concrete `TransactionBuilder` subclass.

## API Schema

```
namespace consensusnode.transactions.spi
requires {Receipt, Response, Transaction, Record} from consensusnode.transactions
requires {MethodDescriptor} from grpc
requires {TransactionBody, TransactionResponse, TransactionReceipt, TransactionRecord} from consensusnode.proto

abstraction TransactionSupport<$$Receipt extends Receipt, $$Transaction extends Transaction<$$Receipt>> {

    type getTransactionType() 

    MethodDescriptor getMethodDescriptor()

    TransactionBody updateBody(transaction:$$Transaction, protoBody:TransactionBody) // updates the proto TransactionBody with the fields from the Transaction

    $$Transaction create(protoBody:TransactionBody) // creates a Transaction from a proto TransactionBody

    Response<$$Receipt> create(protoResponse:TransactionResponse) // creates a Response from a proto TransactionResponse

    $$Receipt create(protoReceipt:TransactionReceipt) // creates a Receipt from a proto TransactionReceipt

    Record<$$Receipt> create(protoRecord:TransactionRecord) // creates a Record from a proto TransactionRecord
}

// factory methods that need to be implemented

@@throws(not-found-error) @@static TransactionSupport getTransactionSupport(transactionType:type) // returns the TransactionSupport for the given transaction type

@@static set<TransactionSupport> getAllTransactionSupports() // returns all TransactionSupport instances
```

## Questions & Comments
