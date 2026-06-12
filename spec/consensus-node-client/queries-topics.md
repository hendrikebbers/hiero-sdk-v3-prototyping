# Topic Queries API

This namespace defines the read-side counterpart to the topic transactions in
[`transactions-topics.md`](transactions-topics.md). One paid query is exposed: the consensus
node charges a fee for serving a topic's metadata snapshot.

## Description

- **`TopicInfoQuery`** extends `PaidQuery<TopicInfo>` — returns the topic's metadata snapshot:
  current admin / submit keys, memo, expiration, auto-renew configuration, and the running
  state of the message stream (`runningHash` + `sequenceNumber`). It does **not** return
  topic messages — those are not stored on the consensus node and must be read from a mirror
  node (see [`mirror-node-topic.md`](../mirror-node-client/mirror-node-topic.md)).

The query extends `PaidQuery`, so it inherits the `maxQueryPayment` knob, the
`getCost(client)` cost-discovery method, and the `PaidQueryResponse<...>` envelope. See
[`queries.md`](queries.md) for the full payment / envelope semantics.

A query against a topic that does not exist (never created, or expired and reaped) fails at
the protocol level — it does not return a `TopicInfo` with a "deleted" flag. This differs
from the file / account info queries, where deleted entities are still returned as a
snapshot with `deleted = true`: the consensus node does not retain a topic's metadata once
the topic is gone.

The `runningHash` + `sequenceNumber` pair lets a caller cheaply verify that a stream of
messages read from a mirror node is consistent with what the consensus node has actually
ordered — without downloading the messages themselves. The running hash is the
SHA-384 chain over all `TopicMessageSubmit`s applied to the topic so far.

## API Schema

```
namespace consensusnode.queries.topics
requires {Address} from ledger
requires {PublicKey} from keys
requires {PaidQuery} from consensusnode.queries

// Full topic metadata snapshot. Returned by TopicInfoQuery.
type TopicInfo {
    @@immutable topicId: Address
    @@immutable @@nullable topicMemo: string                  // short human-readable label
    @@immutable runningHash: bytes                            // SHA-384 over all submitted messages so far
    @@immutable sequenceNumber: int64                         // number of messages submitted so far
    @@immutable expirationTime: zonedDateTime
    @@immutable @@nullable adminKey: PublicKey                // unset → topic is immutable (no update / delete)
    @@immutable @@nullable submitKey: PublicKey               // unset → topic is public (any account may submit)
    @@immutable @@nullable autoRenewPeriod: seconds
    @@immutable @@nullable autoRenewAccount: Address          // pays auto-renewal; if unset, the topic itself pays
}

// Paid query for the metadata snapshot of a topic. Topic *messages* are not returned by
// this query — they live only on mirror nodes (see mirrornode.topic.TopicRepository).
@@finalType
TopicInfoQuery extends PaidQuery<TopicInfo> {
    @@immutable topicId: Address
}
```

## Examples

### Read topic metadata (paid)

```
HieroClient client = ...;

TopicInfo info = new TopicInfoQuery()
    .topicId(topicId)
    .submit(client)
    .value;

int64       messageCount = info.sequenceNumber;
PublicKey   admin        = info.adminKey;          // null if the topic is immutable
PublicKey   submit       = info.submitKey;         // null if the topic is public
```

### Verify a mirror-node message stream against the consensus node

```
TopicInfo info = new TopicInfoQuery()
    .topicId(topicId)
    .submit(client)
    .value;

// `runningHash` and `sequenceNumber` together pin down the exact ordered prefix the
// consensus node has accepted. A mirror node that claims to have seen N messages must
// produce the same running hash after replaying them.
bytes expectedHash      = info.runningHash;
int64 expectedNumberOfMessages = info.sequenceNumber;
```

### Confirm price up-front

```
PaidQuery<TopicInfo> query = new TopicInfoQuery().topicId(topicId);

NativeToken<ANY, ANY> quotedCost = query.getCost(client);
// show quotedCost to the user, wait for approval, then:

TopicInfo info = query.submit(client).value;
```

`getCost(client)` is non-binding — the price on the subsequent `submit(client)` may differ;
combine it with `maxQueryPayment` for a hard ceiling. See
[`queries.md`](queries.md#examples) for the full pattern.

## Questions & Comments

- **`adminKey` / `submitKey` are typed as `PublicKey`, not `Key`.** Same placeholder typing
  as on the write side; see [`transactions-topics.md`](transactions-topics.md) *Questions &
  Comments* for why a single `PublicKey` cannot express the recursive `Key` sum type that
  HAPI actually returns (entries may be `KeyList`, `ThresholdKey`, `ContractID`, …). Tracked
  in [`missing-features.md`](../../missing-features.md) section 3.2 — the `Key` row is a
  prerequisite for retyping both fields.

- **No `deleted` flag, unlike `FileInfo` / `AccountInfo`.** HAPI does not return a deleted
  topic via `ConsensusGetTopicInfo`: once a topic is removed (by `TopicDelete` or by
  expiration), the consensus node drops its metadata and the query fails with
  `INVALID_TOPIC_ID`. The asymmetry with files / accounts is honest to the protocol; callers
  must handle the failure path rather than branching on a flag.

- **No `ledgerId` on `TopicInfo`.** Matches the choice made for `FileInfo` and `AccountInfo`
  — the caller already knows which ledger they queried through their `HieroClient`, and
  ledger-origin metadata belongs on the response envelope, not duplicated on every typed
  payload. The corresponding envelope-level question is opened in
  [`queries.md`](queries.md) *Questions & Comments*.

- **`runningHashVersion` is not exposed.** HAPI's `ConsensusTopicInfo` carries a
  `runningHashVersion` (a small `uint64` identifying the hashing algorithm / formulation used
  to produce `runningHash`) alongside the hash itself. The V3 `TopicInfo` omits it on the
  assumption that V3 only targets ledgers that ship the current (v3) running-hash
  formulation; any future version bump would surface here. If multi-version verification ever
  becomes a real use case, add `@@default(3) runningHashVersion: int32` — additive change.

- **HIP-991 custom-fee fields are out of scope.** `ConsensusGetTopicInfo` in HAPI also carries
  `feeScheduleKey: Key`, `feeExemptKeyList: list<Key>`, and `customFees: list<FixedCustomFee>`
  for revenue-generating topics. These are deliberately omitted here for the same reason
  they are absent on `TopicCreate` / `TopicUpdate` — see
  [`transactions-topics.md`](transactions-topics.md) *Questions & Comments* and
  [`missing-features.md`](../../missing-features.md) sections 1.5 and 3.3. Adding them here
  is additive once the write-side custom-fee model lands.

- **`TopicInfo` here is distinct from `mirrornode.topic.Topic`.** Same conceptual data but
  different sources (consensus node gRPC vs. mirror node REST) and slightly different field
  semantics (e.g. the mirror-node `Topic` carries `createdTimestamp` / `fromTimestamp` /
  `toTimestamp` plus a `deleted` flag because the mirror node retains history, whereas the
  consensus-node `TopicInfo` is a point-in-time snapshot that simply does not exist for
  removed topics). The two are kept separate to avoid coupling the consensus-node API to
  mirror-node update cadence — same split as `AccountInfo` vs. `mirrornode.account.AccountInfo`.
