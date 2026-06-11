# Topic Transactions API

## Description

The topic service (HCS — Hiero Consensus Service) is an ordered, append-only message log
identified by a topic id. Producers submit messages; consensus order across all producers is
established by the consensus node and republished by mirror nodes for subscribers.

Each topic carries two optional keys that control how it can be modified and used:

- **`adminKey`** — required to update or delete the topic. When unset, the topic is
  **immutable**: nobody, not even the original creator, can change its metadata or delete it.
  The topic will then live until its expiration time and be removed by the protocol.
- **`submitKey`** — required to submit messages to the topic. When unset, the topic is
  **public**: any account may submit. When set, only callers who sign with this key may post.

Like accounts, topics expire and can be auto-renewed: `autoRenewPeriod` is the renewal window
the protocol applies when expiration approaches, and `autoRenewAccount` (optional) names the
account that pays the renewal fee. If `autoRenewAccount` is unset, renewal is paid by the
topic itself (from any HBAR it received via HIP-991 custom fees — outside the scope of this
file).

A topic's lifecycle is:

```
  create ── update* ── delete
                  │
                  └── (TopicMessageSubmit, modelled separately)
```

`TopicMessageSubmit` and the HIP-991 custom-fee fields (`customFees`, `feeScheduleKey`,
`feeExemptKeyList`) are deliberately out of scope of this file — see *Questions & Comments*.

### Signing requirements (topic lifecycle)

| Transaction      | Signers required                                                                                              |
|------------------|---------------------------------------------------------------------------------------------------------------|
| `TopicCreate`    | the *payer*; **and** the new `adminKey` if it is set (anti-spoofing: cannot bind another party's key as admin without their consent); **and** the `autoRenewAccount` if it is set and differs from the payer. The `submitKey` does **not** need to sign creation — it gates only future submits, no spoofing risk. |
| `TopicUpdate`    | the *payer*; **and** the topic's *current* `adminKey`; **and**, if `adminKey` itself is being rotated, the *new* key; **and**, if `autoRenewAccount` is set/changed to a different account, that new account |
| `TopicDelete`    | the *payer*; **and** the topic's current `adminKey` (an immutable topic — `adminKey` unset — cannot be deleted; the request fails with `UNAUTHORIZED`) |

When the operator holds every required key, `signWithOperatorAndSubmit(client)` suffices.
Multi-key flows go through the same `signWithOperator(client).sign(...)` pattern shown in
[`transactions-accounts.md`](transactions-accounts.md).

The general rule: a key field on a create transaction must sign **if** binding it would grant
that key authority over a real asset (admin / auto-renew payer / treasury / ...). A key field
that only gates *future* permissioned operations and confers no liability or authority right
now (here: `submitKey`) does not need to sign creation.

### Mutability and "clearing" keys / auto-renew

`TopicUpdate` distinguishes "leave unchanged" from "clear":

- A `@@nullable` field set to **null** (or never set on the builder) means *leave unchanged*.
- Clearing an optional key (e.g. removing `adminKey` to make the topic immutable, or removing
  `submitKey` to make the topic public) is currently modelled by passing an explicit
  *empty-key sentinel*. The exact sentinel depends on the eventual `Key` sum type and is
  flagged in *Questions & Comments* — V2 SDKs use an empty `KeyList`, but this V3 spec does
  not yet have `KeyList`.

## API Schema

```
namespace consensusnode.transactions.topics
requires {Address} from ledger
requires {PublicKey} from keys
requires {Receipt, Transaction} from consensusnode.transactions

// Creates a new topic. Both keys are optional:
//   - adminKey unset  → topic is immutable (no update / delete possible)
//   - submitKey unset → topic is public (anyone may submit messages)
@@finalType
TopicCreateTransaction extends Transaction<TopicCreateReceipt> {
    @@immutable @@nullable adminKey: PublicKey          // controls update / delete; unset → immutable topic; must sign the create transaction when set (anti-spoofing)
    @@immutable @@nullable submitKey: PublicKey         // controls message submission; unset → public topic; does NOT need to sign creation
    @@immutable @@nullable topicMemo: string            // short human-readable label (max 100 chars)
    @@immutable @@nullable autoRenewPeriod: seconds     // protocol default applies when unset
    @@immutable @@nullable autoRenewAccount: Address    // pays auto-renewal; must sign the create transaction if set
}

@@finalType
TopicCreateReceipt extends Receipt {
    @@immutable topicId: Address                        // id of the newly created topic
}

// Updates one or more of a topic's mutable fields. Every nullable field is "leave unchanged"
// when null. Requires the topic's current adminKey to sign; an immutable topic (no adminKey)
// cannot be updated.
@@finalType
TopicUpdateTransaction extends Transaction<TopicUpdateReceipt> {
    @@immutable topicId: Address
    @@immutable @@nullable adminKey: PublicKey          // when set, replaces the admin key; the new key must also sign
    @@immutable @@nullable submitKey: PublicKey         // when set, replaces the submit key
    @@immutable @@nullable topicMemo: string
    @@immutable @@nullable expirationTime: zonedDateTime
    @@immutable @@nullable autoRenewPeriod: seconds
    @@immutable @@nullable autoRenewAccount: Address    // when set, the new account must also sign
}

@@finalType
TopicUpdateReceipt extends Receipt {
}

// Deletes a topic. Only possible if the topic has an adminKey (an immutable topic cannot be
// deleted). The adminKey must sign.
@@finalType
TopicDeleteTransaction extends Transaction<TopicDeleteReceipt> {
    @@immutable topicId: Address
}

@@finalType
TopicDeleteReceipt extends Receipt {
}
```

## Examples

### Create a public, immutable topic

No `adminKey`, no `submitKey`. Anyone may submit messages; no one — not even the creator —
can ever change the topic's metadata or delete it. Useful for cases where the operator wants
an unalterable audit log.

```
HieroClient client = ...;

Response<TopicCreateReceipt> response = new TopicCreateTransaction()
    .topicMemo("audit-log")
    .signWithOperatorAndSubmit(client);

Address topicId = response.queryReceipt().topicId;
```

### Create a private, mutable topic (operator controls both keys)

```
new TopicCreateTransaction()
    .adminKey(client.operatorPublicKey)
    .submitKey(client.operatorPublicKey)
    .topicMemo("ops-events")
    .signWithOperatorAndSubmit(client);
```

### Rotate the admin key (multi-signature)

Both the *current* and the *new* admin key must sign, in addition to the payer.

```
PackedTransaction<...> packed = new TopicUpdateTransaction()
    .topicId(topicId)
    .adminKey(newAdminPublicKey)
    .signWithOperator(client)        // payer + current adminKey (operator)
    .sign(newAdminSigner);           // new adminKey

packed.submit(client);
```

### Delete a topic

```
new TopicDeleteTransaction()
    .topicId(topicId)
    .signWithOperatorAndSubmit(client);
```

## Questions & Comments

- **`adminKey` / `submitKey` are typed as `PublicKey`, not `Key`.** Same placeholder typing
  as in [`transactions-files.md`](transactions-files.md): HAPI takes a `Key` (sum type over
  `PublicKey` / `ContractID` / `KeyList` / `ThresholdKey`), so the current single-key shape
  cannot express m-of-n custody, contract-controlled topics, or nested key structures. The
  `Key` sum type is tracked in [`missing-features.md`](../../missing-features.md) section
  2.2 as the prerequisite; once it lands, both fields should be retyped to `Key`.

- **Clearing `adminKey` / `submitKey` / `autoRenewAccount` on update — partial coverage.**
  HAPI distinguishes "leave unchanged" from "remove" via sentinel values that the consensus
  node interprets server-side:
  - For `Address`-typed fields (`autoRenewAccount`): the sentinel is `0.0.0`, exposed as
    [`ZERO_ADDRESS`](../base/ledger.md) in the `ledger` namespace. Pass `ZERO_ADDRESS` to
    `autoRenewAccount(...)` on an update to drop the auto-renew account.
  - For `Key`-typed fields (`adminKey`, `submitKey`): the sentinel is an empty `KeyList`,
    which the V3 spec cannot yet express because `Key` / `KeyList` are still missing (see
    [`missing-features.md`](../../missing-features.md) section 2.2). Until those land,
    there is no way to e.g. turn a private topic public via update or freeze an updatable
    topic by removing its admin key. Once `Key` exists, an explicit `Key.EMPTY` value
    (mapping to `KeyList{keys: []}`) is the natural counterpart to `ZERO_ADDRESS`.

  V2 SDKs additionally expose dedicated `clearAutoRenewAccountId()` / `clearAdminKey()` /
  `clearSubmitKey()` methods that hide the sentinel from callers. V3 could later layer the
  same convenience methods over the sentinels (additive change) — see the placement
  decision recorded in the `ZERO_ADDRESS` comment in `ledger.md`.

- **`TopicMessageSubmit` is out of scope of this file.** Message-submit shares the chunking
  question raised in [`transactions-files.md`](transactions-files.md) (`FileAppend` chunked
  appends) and additionally interacts with HIP-991 custom-fee limits; it deserves its own
  spec rather than being smuggled into the lifecycle file. Tracked in
  [`missing-features.md`](../../missing-features.md) section 1.5.

- **HIP-991 custom-fee fields are out of scope.** `TopicCreate` / `TopicUpdate` in HAPI
  carry `customFees: list<FixedCustomFee>`, `feeScheduleKey: Key`, and
  `feeExemptKeyList: list<Key>` for revenue-generating topics. These are tracked in
  [`missing-features.md`](../../missing-features.md) sections 1.5 and 2.3 and depend on a
  write-side custom-fee model that does not yet exist in the spec. Adding them here is
  additive and breaks nothing.

