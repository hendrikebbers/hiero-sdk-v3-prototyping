# Mirror Node Network Query API

## Description

Network-wide read queries: exchange rates, the current fee schedule (`fees`), HIP-1313 fee
estimation (`estimateFees`), staking parameters, HBAR supply, and the consensus-node address book
(`nodes`).

## API Schema

```
namespace mirrornode.network
requires {AccountId, MirrorNode} from ledger
requires {Authority} from authority
requires {Page} from common

@@finalType
ExchangeRate {
    @@immutable centEquivalent: int32
    @@immutable hbarEquivalent: int32
    @@immutable expirationTime: zonedDateTime
}

@@finalType
ExchangeRates {
    @@immutable currentRate: ExchangeRate
    @@immutable nextRate: ExchangeRate
}

@@finalType
NetworkFee {
    @@immutable gas: int64
    @@immutable transactionType: string
}

@@finalType
NetworkStake {
    @@immutable maxStakeReward: int64
    @@immutable maxStakeRewardPerHbar: int64
    @@immutable maxTotalReward: int64
    @@immutable nodeRewardFeeFraction: double
    @@immutable reservedStakingRewards: int64
    @@immutable rewardBalanceThreshold: int64
    @@immutable stakeTotal: int64
    @@immutable stakingPeriod: seconds
    @@immutable stakingPeriodsStored: int64
    @@immutable stakingRewardFeeFraction: double
    @@immutable stakingRewardRate: int64
    @@immutable stakingStartThreshold: int64
    @@immutable unreservedStakingRewardBalance: int64
}

@@finalType
NetworkSupplies {
    @@immutable releasedSupply: int256
    @@immutable totalSupply: int256
}

// A reachable gRPC endpoint of a consensus node (GET /network/nodes). One of ipAddressV4 /
// domainName is populated (HIP-1046 added domain names).
@@finalType
ServiceEndpoint {
    @@immutable @@nullable ipAddressV4: string        // dotted-quad IPv4, when published
    @@immutable port: int32
    @@immutable @@nullable domainName: string         // DNS name, when published (HIP-1046)
}

// One consensus-node entry in the address book. Returned by NetworkRepository.nodes()
// (GET /network/nodes).
@@finalType
NetworkNode {
    @@immutable nodeId: int64                          // stable HIP-869 node id
    @@immutable accountId: AccountId                   // fee account that receives this node's share
    @@immutable @@nullable description: string
    @@immutable @@nullable publicKey: string           // node gossip public key (hex)
    @@immutable @@nullable nodeCertHash: string        // SHA-384 of the node's TLS certificate (hex)
    @@immutable @@default([]) serviceEndpoints: list<ServiceEndpoint>
    @@immutable @@nullable adminKey: Authority         // DAB admin key (HIP-869); null on pre-DAB entries
    @@immutable stake: int64                           // total stake delegated to this node (tinybars)
    @@immutable @@nullable timestamp: zonedDateTime    // consensus time of this address-book entry
}

// Estimated fee for a candidate transaction (HIP-1313, congestion-adjusted). Returned by
// NetworkRepository.estimateFees(). PROVISIONAL: the authoritative HIP-1313 response model
// (full breakdown + high-volume multiplier) is tracked in missing-features.md §3.4; only the
// estimated total is modelled here for now.
@@finalType
FeeEstimate {
    @@immutable estimatedFee: int64                    // estimated total fee in tinybars
}

NetworkRepository {
    @@async @@throws(mirror-node-error)
    @@nullable ExchangeRates exchangeRates()

    @@async @@throws(mirror-node-error)
    list<NetworkFee> fees()

    @@async @@throws(mirror-node-error)
    @@nullable NetworkStake stake()

    @@async @@throws(mirror-node-error)
    @@nullable NetworkSupplies supplies()

    // The network's consensus-node list (address book).
    // Maps to GET /api/v1/network/nodes.
    @@async @@throws(mirror-node-error)
    Page<NetworkNode> nodes()

    // Estimate the fee for a protobuf-encoded candidate transaction (HIP-1313). The transaction
    // is passed as encoded bytes — the mirror-node layer does not depend on the consensus-client
    // transaction types. Maps to POST /api/v1/network/fees.
    @@async @@throws(mirror-node-error)
    FeeEstimate estimateFees(transaction: bytes)
}

@@static NetworkRepository createRepository(mirrorNode: MirrorNode)

```

## Questions & Comments

- **`fees()` vs `estimateFees()`.** `fees()` (existing) returns the network's *current* per-type
  gas fee schedule (`GET /network/fees`). `estimateFees()` is the HIP-1313 *estimation* of a
  specific candidate transaction (`POST /network/fees`) — a different operation despite sharing the
  path.
- **`FeeEstimate` is provisional.** The OpenAPI references `FeeEstimateResponse` but does not expose
  its field set, and the full HIP-1313 fee-estimation model (node / network / service breakdown
  plus the high-volume congestion multiplier) is tracked in
  [`missing-features.md`](../../missing-features.md) §3.4. Only `estimatedFee` is modelled here;
  expand once §3.4 lands the authoritative shape.
- **`estimateFees(transaction: bytes)` takes encoded bytes, not a typed transaction.** The
  mirror-node layer is independent of `consensus-node-client`, so it cannot reference
  `PackedTransaction`; the protobuf-encoded transaction bytes are the layering-clean input (and
  match what the REST endpoint accepts).
- **`NetworkNode` is the mirror-node read shape**, distinct from the consensus-node
  `NodeAddress` returned by `NodeAddressBookQuery`
  ([`queries-network.md`](../consensus-node-admin-client/queries-network.md)) — same entity, two
  sources, slightly different fields. `ServiceEndpoint` is redefined locally here (the mirror node
  cannot import the consensus-admin `ServiceEndpoint`).
- **`NetworkNode.adminKey` is an `Authority`.** The mirror returns the DAB admin key as a `Key`;
  V3 types it as the authorization `Authority` (per [ADR-0004](../../docs/adr/0004-authority-authorization-sum-type.md)).