# Ledger API

## Description

## API Schema

```
namespace ledger
requires {NativeTokenUnit} from nativeToken

// Represents a specific ledger instance
Ledger<$$Unit extends NativeTokenUnit> {
    @@immutable id: bytes // identifier of the ledger
    @@immutable @@nullable name: string // human readable name of the network
    @@immutable nativeTokenUnit: $$Unit
}

// Represents the base of an address on a network.
Address {
    @@immutable shard: uint64 // shard number
    @@immutable realm: uint64 // realm number
    @@immutable num: uint64 // account number
    @@immutable checksum: string // checksum of the address
    
    // Validates the checksum of the address
    bool validateChecksum(ledger: Ledger)
    
    // returns address in format "shard.realm.num"
    string toString()
    
    // returns address in format "shard.realm.num-checksum"
    string toStringWithChecksum()
}

// Id of a transaction
abstraction TransactionId {
  @@immutable accountId:Address // the account that is the payer of the transaction
  @@immutable validStart:zonedDateTime // the start time of the transaction
  @@immutable @@nullable nonce:int32 // nonce of an internal transaction

  string toString() // returns the transaction id as a string
  string toStringWithChecksum() // returns the transaction id as a string with a checksum
}

// Represents a consensus node on a network.
ConsensusNode {
    @@immutable ip: string // ip address of the node
    @@immutable port: uint16 // port of the node
    @@immutable Address account // account of the node
}

// Represents a mirror node on a network.
MirrorNode {
    @@immutable restBaseUrl: string // base url of the mirror node REST API (scheme://host[:port]/api/v1)
}

// factory methods of Address that should be added to the namespace in the best language dependent way

// Parses Address from string format: "shard.realm.num" or "shard.realm.num-checksum"
// @@throws(illegal-format) if format is invalid, values are negative, or parsing fails
// Supports optional checksum suffix after dash
@@throws(illegal-format) @@static Address fromString(address: string)

// Factory methods for TransactionId
@@static TransactionId generateTransactionId(accountId:Address)
@@throws(illegal-format) @@static TransactionId fromString(transactionId:string)

// Factory methods for transaction loading
@@static Transaction<$$Receipt extends Receipt> fromBytes(transactionBytes: bytes)
```

## Questions & Comments

- [@hendrikebbers](https://github.com/hendrikebbers): Should we rename `Ledger` to `Network`?