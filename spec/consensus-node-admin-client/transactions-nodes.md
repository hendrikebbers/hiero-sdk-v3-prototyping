# Dynamic Address Book Node Transaction API

## Description

The **Dynamic Address Book** (HIP-869, with the HIP-1046 follow-up) lets the governance
council register, update, and remove consensus nodes via on-network transactions instead of
static JSON files baked into a release. Each consensus node is identified by a numeric
`nodeId` and is bound to:

- An **`accountId`** that receives the node's staking rewards.
- A **gossip endpoint set** (`gossipEndpoints`) — internal endpoints used for hashgraph
  gossip between consensus nodes. Not reachable by clients.
- A **service endpoint set** (`serviceEndpoints`) — public gRPC endpoints that clients use
  to submit transactions.
- A **gossip CA certificate** (`gossipCaCertificate`) — the X.509 certificate (DER bytes)
  used to terminate mutual TLS on the gossip channel.
- A **gRPC certificate hash** (`grpcCertificateHash`) — SHA-384 of the certificate served
  on the public gRPC endpoints; clients pin against this hash.
- An **`adminKey`** — required to update or delete the node entry. There is no system-key
  override here; rotating the admin key requires both the old and new keys.
- A **`declineReward`** flag — when `true`, the node opts out of receiving staking rewards.
- An optional **`grpcWebProxyEndpoint`** (HIP-1046) — a gRPC-web proxy endpoint that
  browser clients can use without their own TLS terminator.

A node's lifecycle is:

```
  create ── update* ── delete
```

`NodeDelete` does not free the `nodeId`; deleted nodes remain in the address book in a
tombstoned state so that historical references still resolve.

### Signing requirements (DAB)

| Transaction               | Signers required                                                                                                                                              |
|---------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `NodeCreateTransaction`   | the *payer*; **and** the address-book *council* signer; **and** the new `adminKey` (anti-spoofing); **and** the `accountId`'s key (so the council cannot bind another party's account to a node) |
| `NodeUpdateTransaction`   | the *payer*; **and** the node's current `adminKey`; **and**, if `adminKey` itself is being rotated, the *new* key; **and**, if `accountId` is changed, the new account's key |
| `NodeDeleteTransaction`   | the *payer*; **and** the node's current `adminKey`                                                                                                            |

The council signer is configured at the consensus-node level and reaches the SDK through
`TransactionSigner` like any other externally held key.

### `nodeId` vs. the read-side `ConsensusNode.account`

The DAB transactions identify a node by its sequential **`nodeId: int64`** — the same
identifier the consensus layer uses internally for gossip and stake distribution.
[`base/ledger.md`](../base/ledger.md) separately defines a `ConsensusNode` type with an
**`account: AccountId`** field. Both refer to the same node, from two different angles:

- `account` is the node's **fee account** (where the per-transaction fee share is paid).
  Clients select a node for transaction routing by its `account`; the value may change
  when the operator rotates the reward account via `NodeUpdate.accountId`.
- `nodeId` is the node's **stable network identifier**. It is assigned at `NodeCreate`,
  returned in `NodeCreateReceipt.nodeId`, and remains constant for the lifetime of the
  node — independent of account rotation, endpoint migration, or certificate rollover.

The same dichotomy already appears in
[`transactions-accounts.md`](../consensus-node-client/transactions-accounts.md), where
`stakedAccountId: AccountId` and `stakedNodeId: int64` are modelled as mutually exclusive
alternatives. The DAB transactions take only `nodeId`, because HAPI's address-book service
addresses nodes purely by their stable identifier (account rotation must not break a
pending `NodeUpdate`). `ConsensusNode` does **not** currently carry `nodeId` — see
*Questions & Comments* for why and what an extension would look like.

### ServiceEndpoint addressing

Each `ServiceEndpoint` carries either an `IpAddress` **or** a domain name (mutually
exclusive) plus a port:

- `ipAddress` — an `IpAddress` value (defined in [`base/ledger.md`](../base/ledger.md)).
  IPv4-only today (the wire shape is HAPI `ipAddressV4`, 4 bytes); the type is named
  `IpAddress` rather than `IpV4Address` so adding IPv6 later only requires loosening the
  type's byte-length constraint, not renaming the field.
- `domainName` — DNS name, used when the operator runs behind a load balancer, wants to
  decouple addressing from a specific IP, or needs IPv6 reachability today (via an AAAA
  record on the resolved name).

The consensus node validates the `@@oneOf` at submission time; setting both or neither
fails with `INVALID_ENDPOINT`.

## API Schema

```
namespace consensusnode.admin.nodes
requires {AccountId, IpAddress} from ledger
requires {PublicKey} from keys
requires {Receipt, Transaction} from consensusnode.transactions

// One reachable endpoint on a consensus node. Exactly one of ipAddress or domainName must
// be set; the network rejects an endpoint with both (or neither) as INVALID_ENDPOINT.
@@oneOf(ipAddress, domainName)
type ServiceEndpoint {
    @@immutable @@nullable ipAddress: IpAddress   // IPv4 today (HAPI ipAddressV4 wire shape); null when domainName is used
    @@immutable @@nullable domainName: string     // DNS name; null when ipAddress is used
    @@immutable port: uint16                      // listening port (0 is invalid)
}

// Registers a new consensus node in the address book. All endpoint, certificate, and key
// fields are required; the network has no defaults for any of them.
@@finalType
NodeCreateTransaction extends Transaction<NodeCreateReceipt> {
    @@immutable accountId: AccountId                                     // account that receives the node's staking rewards
    @@immutable @@nullable description: string                           // free-form description (max 100 chars)
    @@immutable @@minLength(1) gossipEndpoints: list<ServiceEndpoint>    // inter-node hashgraph gossip endpoints
    @@immutable @@minLength(1) serviceEndpoints: list<ServiceEndpoint>   // public gRPC endpoints for client transactions
    @@immutable gossipCaCertificate: bytes                               // X.509 DER bytes of the certificate that terminates mutual TLS for gossip
    @@immutable grpcCertificateHash: bytes                               // SHA-384 of the certificate served on serviceEndpoints; clients pin against this
    @@immutable adminKey: PublicKey                                      // required to update / delete the node; must co-sign this create
    @@immutable @@default(false) declineReward: bool                     // true → node opts out of staking rewards
    @@immutable @@nullable grpcWebProxyEndpoint: ServiceEndpoint         // optional gRPC-web proxy endpoint (HIP-1046)
}

@@finalType
NodeCreateReceipt extends Receipt {
    @@immutable nodeId: int64                                            // network-assigned numeric id of the new node
}

// Updates one or more of a node's mutable fields. Every nullable field is "leave unchanged"
// when null. Lists are replace-on-set: passing a non-null endpoint list replaces the entire
// current list (there is no per-entry append / remove).
@@finalType
NodeUpdateTransaction extends Transaction<NodeUpdateReceipt> {
    @@immutable nodeId: int64
    @@immutable @@nullable accountId: AccountId                          // when set, the new account's key must also sign
    @@immutable @@nullable description: string
    @@immutable @@nullable gossipEndpoints: list<ServiceEndpoint>        // replaces the entire list when set
    @@immutable @@nullable serviceEndpoints: list<ServiceEndpoint>       // replaces the entire list when set
    @@immutable @@nullable gossipCaCertificate: bytes
    @@immutable @@nullable grpcCertificateHash: bytes
    @@immutable @@nullable adminKey: PublicKey                           // when set, the new key must also sign
    @@immutable @@nullable declineReward: bool
    @@immutable @@nullable grpcWebProxyEndpoint: ServiceEndpoint
}

@@finalType
NodeUpdateReceipt extends Receipt {
}

// Tombstones a node entry. The nodeId is not freed; subsequent lookups return a deleted
// flag rather than NOT_FOUND so historical references resolve.
@@finalType
NodeDeleteTransaction extends Transaction<NodeDeleteReceipt> {
    @@immutable nodeId: int64
}

@@finalType
NodeDeleteReceipt extends Receipt {
}
```

## Examples

### Register a new consensus node

```
HieroClient        client       = ...;   // council operator; signer wraps council HSM
Account            nodeOperator = ...;   // owns the new accountId, must co-sign
TransactionSigner  adminSigner  = ...;   // wraps the new node's adminKey

PackedTransaction<...> packed = new NodeCreateTransaction()
    .accountId(nodeOperator.accountId)
    .description("consensus-node-7")
    .gossipEndpoints([
        new ServiceEndpoint(IpAddress.fromString("10.0.0.7"), /* domainName */ null, 50211),
    ])
    .serviceEndpoints([
        new ServiceEndpoint(null, "node7.consensus.example", 50211),
    ])
    .gossipCaCertificate(gossipCertDer)
    .grpcCertificateHash(grpcCertSha384)
    .adminKey(newAdminPublicKey)
    .signWithOperator(client)        // council + payer
    .sign(nodeOperator)              // accountId binding
    .sign(adminSigner);              // adminKey binding (anti-spoofing)

int64 nodeId = packed.submit(client).queryReceipt().nodeId;
```

### Update a node's service endpoints

```
new NodeUpdateTransaction()
    .nodeId(nodeId)
    .serviceEndpoints([
        new ServiceEndpoint(null, "node7-new.consensus.example", 50211),
    ])
    .signWithOperator(client)        // payer
    .sign(adminSigner);              // current adminKey
```

### Rotate the admin key

```
PackedTransaction<...> packed = new NodeUpdateTransaction()
    .nodeId(nodeId)
    .adminKey(newAdminPublicKey)
    .signWithOperator(client)        // payer
    .sign(currentAdminSigner)        // current adminKey
    .sign(newAdminSigner);           // new adminKey

packed.submit(client);
```

### Delete a node

```
new NodeDeleteTransaction()
    .nodeId(nodeId)
    .signWithOperator(client)
    .sign(adminSigner);
```

## Questions & Comments

- **`ConsensusNode` in [`base/ledger.md`](../base/ledger.md) does not carry `nodeId`
  today.** It models the routing / fee-account view (`ip`, `port`, `account: AccountId`),
  which predates HIP-869 and is sufficient for clients that only need to *talk to* a node.
  The DAB transactions on the other hand need the *stable* identifier that survives an
  account rotation, hence `int64 nodeId`. Resolving the asymmetry on the read side
  (`ConsensusNode { nodeId, ip, port, account }`) is an additive change deliberately
  deferred: existing `ConsensusNode` consumers are not broken by leaving the field out, and
  the right time to introduce it is together with the `NodeId` type from §3.1 of
  `missing-features.md` — adding an untyped `int64` first and then retyping later would
  churn every call site twice. Until then, callers that need to bridge the two worlds
  remember the `nodeId` returned by `NodeCreateReceipt` alongside the `Address` they obtain
  from the address-book lookup.

- **`gossipCaCertificate` and `grpcCertificateHash` are raw `bytes`.** The encoding rules
  (X.509 DER, SHA-384) are documented in the field comments but not enforced by the type
  system. A future `X509Certificate` / `CertificateFingerprint` pair in the `keys`
  namespace could make the encoding explicit, in line with how PKCS#8 / SPKI / PEM are
  handled there today.

- **`ServiceEndpoint.ipAddress` is typed as `IpAddress` (IPv4-only today).** The
  byte-length invariant is enforced by [`IpAddress`](../base/ledger.md) itself
  (`@@minLength(4) @@maxLength(4)`), not by server-side validation only. The type is
  intentionally named `IpAddress` rather than `IpV4Address`: once a HIP adds IPv6 to the
  consensus-node wire shape, only the `@@maxLength` constraint relaxes to `16` and every
  existing `IpAddress`-typed call site automatically supports both. Until that HIP exists,
  IPv6 reachability is achieved through `domainName` + a DNS AAAA record on the resolved
  host.

- **Lists are replace-on-set on update.** Matches HAPI but is a common foot-gun (callers
  expecting "update" to mean "add one endpoint" will silently drop the rest of the list).
  Consider whether a language-binding-level convenience that reads the current list,
  mutates a copy, and writes it back belongs in the enterprise layer.

- **No HIP-1046 multi-endpoint gRPC-web proxy field.** `grpcWebProxyEndpoint` is a single
  endpoint, matching the current HAPI shape; if HIP-1046 evolves to a list, both
  `NodeCreate` and `NodeUpdate` need to grow accordingly. Tracked here for visibility.
