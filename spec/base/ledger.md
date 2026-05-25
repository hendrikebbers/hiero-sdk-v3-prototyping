# Ledger API

## Description

## API Schema

```
namespace ledger

// Represents a specific ledger instance
Ledger {
    @@immutable id: bytes // identifier of the ledger
    @@immutable @@nullable name: string // human readable name of the network
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
```

## Questions & Comments

- [@hendrikebbers](https://github.com/hendrikebbers): Should we rename `Ledger` to `Network`?