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
    bool validateChecksum(ledger: Ledger<ANY>)
    
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

// Single IP address representation, stored as raw network-order bytes. Today the type
// only accepts IPv4 (exactly 4 bytes), matching the HAPI consensus-node wire shape
// (`ServiceEndpoint.ipAddressV4`, which is IPv4-only). The type is intentionally named
// `IpAddress` (not `IpV4Address`) so that adding IPv6 support later is purely additive:
// a future HIP that adds IPv6 to the wire shape only needs to relax the @@maxLength
// constraint to 16; every call site that takes `IpAddress` keeps working unchanged. Until
// then, IPv6 reachability for a node is achieved via a domain name in
// `consensusnode.admin.nodes.ServiceEndpoint.domainName` whose DNS AAAA record resolves to
// the v6 address.
type IpAddress {
    @@immutable @@minLength(4) @@maxLength(4) bytes: bytes   // 4 bytes, network byte order (IPv4); constraint widens to allow 16 bytes once IPv6 lands

    // Dotted-quad form ("10.0.0.7") today; once IPv6 lands, this returns the RFC 5952
    // canonical form for 16-byte values.
    string toString()
}

// Parses an IpAddress from its textual form. Today only dotted-quad IPv4 ("10.0.0.7") is
// accepted; everything else throws illegal-format.
@@throws(illegal-format) @@static IpAddress IpAddress.fromString(value: string)

// Wraps raw network-order bytes. `value.length` must equal 4; otherwise throws
// illegal-format. Loosens to {4, 16} when IPv6 support is added.
@@throws(illegal-format) @@static IpAddress IpAddress.fromBytes(value: bytes)

// Represents a consensus node on a network. This is the routing / fee view: clients use
// (ip, port) to reach the node and `account` to identify where its transaction fees flow.
// It is intentionally distinct from the DAB `nodeId: int64` (HIP-869) that
// `consensusnode.admin.nodes` transactions use to identify a node for create / update /
// delete — `account` can be rotated by NodeUpdate while `nodeId` is stable across the
// node's lifetime. Carrying `nodeId` here is deferred until the typed-identifier roll-out
// (see missing-features.md §3.1); see also the *Questions & Comments* in
// consensus-node-admin-client/transactions-nodes.md.
ConsensusNode {
    @@immutable ip: IpAddress // ip address of the node (IPv4 today; extensible to IPv6)
    @@immutable port: uint16 // port of the node
    @@immutable account: Address // fee account of the node (NOT the stable DAB nodeId)
}

// Represents a mirror node on a network.
MirrorNode {
    @@immutable restBaseUrl: string // base url of the mirror node REST API (scheme://host[:port]/api/v1)
}

// The zero address (0.0.0). HAPI uses this value as a clear-sentinel on update
// transactions for Address-typed fields: writing ZERO_ADDRESS to e.g.
// TopicUpdateTransaction.autoRenewAccount or AccountUpdateTransaction.stakedAccountId
// removes the previously configured value (distinct from leaving the field at its default
// "unchanged"). The consensus node interprets the sentinel server-side; SDKs do not invent
// the semantic locally. Callers should not use this value as an actual account id — there
// is no account 0.0.0 on the network.
constant ZERO_ADDRESS: Address = Address{shard: 0, realm: 0, num: 0, checksum: ""}

// factory methods of Address that should be added to the namespace in the best language dependent way

// Parses Address from string format: "shard.realm.num" or "shard.realm.num-checksum"
// @@throws(illegal-format) if format is invalid, values are negative, or parsing fails
// Supports optional checksum suffix after dash
@@throws(illegal-format) @@static Address fromString(address: string)

// Factory methods for TransactionId
@@static TransactionId generateTransactionId(accountId:Address)
@@throws(illegal-format) @@static TransactionId fromString(transactionId:string)

```

## Questions & Comments

- [@hendrikebbers](https://github.com/hendrikebbers): Should we rename `Ledger` to `Network`?