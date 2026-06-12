# Network Queries API

## Description

This namespace defines the admin-side read queries against the consensus node:

- **`NetworkVersionInfoQuery`** — free query returning the network's HAPI protocol version
  and the Hedera-services build version. Used by tooling that needs to gate behaviour on a
  specific protocol revision (e.g. probing whether a HIP is live on the targeted network).
  Maps to HAPI's free `NetworkGetVersionInfoQuery` on the `NetworkService`.

- **`NodeAddressBookQuery`** — paid query returning the current address book (the list of
  consensus nodes with their accounts, endpoints, certificates, and stable node ids).
  Useful for clients that want to refresh their network topology without going through the
  mirror node. Under the hood the query reads the network's address-book file
  (file `0.0.102` on default Hedera shards/realms) via the file service; this is a paid
  read, so the query inherits `PaidQuery` semantics (cost discovery, `maxQueryPayment`
  ceiling, operator-paid by default).

Both queries are admin-flavoured in the sense that ordinary applications normally do not
issue them — `HieroClient` already exposes the version and node list via its
`ledger.networkSetting()` snapshot. They land in `consensus-node-admin-client` rather than
in the main `consensus-node-client` because operator / tooling code that *does* need a live
snapshot is the same audience that runs the DAB transactions, and pulling the dependency
on the admin module is acceptable there.

### Why `NodeAddressBookQuery` is `PaidQuery`, not `Query`

HAPI does not have a dedicated free "address book" query on the network service. The
canonical way to read the address book is via `FileGetContents` on file `0.0.102`, which is
a paid `FileService` query. `NodeAddressBookQuery` is a typed convenience that hides the
file mechanism but **not** the cost: the underlying file read is still billed by the
network, so the query honestly extends `PaidQuery` with full cost-discovery behaviour. A
caller that wants a free node list today uses the mirror node (`GET /network/nodes` —
tracked in [`missing-features.md`](../../missing-features.md) §4.2) instead.

### Reusing `ServiceEndpoint` from `consensusnode.admin.nodes`

A `NodeAddress` returned by `NodeAddressBookQuery` exposes its reachable endpoints as
`list<ServiceEndpoint>`, importing the same `ServiceEndpoint` type that
[`transactions-nodes.md`](transactions-nodes.md) defines for the write side. The shape is
identical — IPv4 (via `IpAddress`) or domain name, plus a port — and keeping it as one
type avoids a duplicate read/write shape that would have to evolve in lockstep.

## API Schema

```
namespace consensusnode.admin.network
requires {Address} from ledger
requires {Query, PaidQuery} from consensusnode.queries
requires {ServiceEndpoint} from consensusnode.admin.nodes

// A semantic version triple (major.minor.patch) with optional pre-release / build
// metadata labels, as defined by semver.org. Defined inline here for now; once the
// missing-features §3.1 SemanticVersion lands in `base/common`, this local definition
// should be removed and the base type imported instead.
type SemanticVersion {
    @@immutable major: int32
    @@immutable minor: int32
    @@immutable patch: int32
    @@immutable @@nullable preReleaseLabel: string   // semver "-rc.1" etc.; null when absent
    @@immutable @@nullable buildMetadata: string     // semver "+exp.sha.5114f85" etc.; null when absent

    // Canonical semver string, e.g. "0.49.1-rc.1+exp.sha.5114f85"
    string toString()
}

// Combined version snapshot: the HAPI protocol revision the network is speaking and the
// Hedera-services build that implements it. Returned by NetworkVersionInfoQuery.
type NetworkVersionInfo {
    @@immutable hapiVersion: SemanticVersion        // HAPI / protobuf protocol revision
    @@immutable servicesVersion: SemanticVersion    // Hedera-services build that hosts that protocol
}

// Free query that returns the network's current HAPI and services version. Has no input
// fields — the answer is per-network, not per-entity.
@@finalType
NetworkVersionInfoQuery extends Query<NetworkVersionInfo> {
}

// One address-book entry. Carries the stable DAB identifier (`nodeId`) plus everything a
// client needs to reach the node and authenticate its TLS material. Mirrors the
// write-side `NodeCreateTransaction` shape but is the read-only return form.
type NodeAddress {
    @@immutable nodeId: int64                                // stable HIP-869 identifier; survives account / endpoint rotation
    @@immutable accountId: Address                           // fee account that receives this node's per-transaction share
    @@immutable @@default([]) serviceEndpoints: list<ServiceEndpoint>   // public gRPC endpoints (clients submit transactions here)
    @@immutable @@nullable description: string               // free-form description set at NodeCreate (max 100 chars)
    @@immutable gossipCaCertificate: bytes                   // X.509 DER bytes of the certificate that terminates mutual TLS for gossip
    @@immutable grpcCertificateHash: bytes                   // SHA-384 of the certificate served on serviceEndpoints; clients pin against this
    @@immutable rsaPublicKey: bytes                          // legacy gossip RSA public key (DER); kept for backwards compatibility with pre-DAB tooling
}

// Snapshot of the entire address book at consensus time. Order matches HAPI: nodes are
// listed by ascending `nodeId`.
type NodeAddressBook {
    @@immutable nodes: list<NodeAddress>
}

// Paid query that returns the current address book. The implementation reads the
// network's address-book file (HAPI file 0.0.102 on default Hedera shards/realms) via
// the file service; the operator pays the FileContents query fee. Has no input fields —
// the answer is per-network.
@@finalType
NodeAddressBookQuery extends PaidQuery<NodeAddressBook> {
}
```

## Examples

### Read the network version (free)

```
HieroClient client = ...;

NetworkVersionInfo info = new NetworkVersionInfoQuery()
    .submit(client)
    .value;

SemanticVersion hapi     = info.hapiVersion;
SemanticVersion services = info.servicesVersion;
```

No cost discovery, no `maxQueryPayment` — `NetworkVersionInfoQuery` extends `Query`, not
`PaidQuery`.

### Refresh the address book (paid)

```
PaidQueryResponse<NodeAddressBook> response = new NodeAddressBookQuery()
    .maxQueryPayment(NativeToken.of(...))   // bound the spend
    .submit(client);

NodeAddressBook book = response.value;
NativeToken<ANY, ANY> paid = response.cost; // what was actually charged

for (NodeAddress n : book.nodes) {
    log("node " + n.nodeId + " @ account " + n.accountId.toString());
}
```

The operator pays. A configurable payer (paymaster / custodial / sponsored reads) is
deferred — see
[ADR-0002](../../docs/adr/0002-defer-paid-query-payer-customization.md).

### Gate behaviour on a HIP-availability check

```
NetworkVersionInfo info = new NetworkVersionInfoQuery().submit(client).value;

if (info.servicesVersion.major >= 0 && info.servicesVersion.minor >= 50) {
    // HIP-1046 grpcWebProxyEndpoint is supported on this network
    new NodeUpdateTransaction()
        .nodeId(nodeId)
        .grpcWebProxyEndpoint(...)
        .signWithOperatorAndSubmit(client);
}
```

## Questions & Comments

- **`SemanticVersion` is defined inline.** The same type appears in v2 SDKs across several
  service boundaries (network version, services version, future HIP-feature checks) and
  belongs in `base/common`. It is defined locally here so this file is self-contained, but
  once [`missing-features.md`](../../missing-features.md) §3.1 lands a base `SemanticVersion`
  this local definition should be removed and the base type imported.

- **`NodeAddressBookQuery` is paid because it goes through the file service.** The
  underlying HAPI mechanism is `FileGetContents` on file `0.0.102`, which the network bills.
  Modelling it as a free query would lie about the cost. A future protocol change that
  introduces a dedicated free network-service query for the address book (no current HIP)
  would let this be retyped to `Query`, but that is a breaking type-level change by the
  same free/paid promotion rule as for any other query.

- **`NodeAddress.rsaPublicKey` is pre-DAB legacy.** The gossip layer authenticates via the
  `gossipCaCertificate` X.509 chain in modern HAPI; the RSA key is retained for tooling
  that still reads the legacy address-book format. New code should rely on
  `gossipCaCertificate` + `grpcCertificateHash`. A future HIP that drops the field would
  let it be removed cleanly (breaking change for anyone still reading it; nothing the
  consensus node would notice).

- **Result type returns an empty `nodeId: 0` field?** When a node is deleted via
  `NodeDeleteTransaction`, HAPI keeps the entry in the file under a tombstone flag, but
  this query's `NodeAddressBook` does not surface a `deleted` boolean per entry. Whether
  tombstoned entries should be visible (and how) is open — today they are filtered out
  client-side by the SDK before returning the typed result.

- **`NodeAddressBookQuery` has no input fields.** Address books are per-network, not
  per-shard / per-realm — a single network has one address book file. Multi-network
  callers issue this query once per `HieroClient` they hold.
