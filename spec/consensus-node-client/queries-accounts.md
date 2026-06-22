# Account Queries API

This namespace defines account-related queries against the consensus node. It is the first
concrete usage of the abstractions defined in [`queries.md`](queries.md) and serves as a
worked example of the free / paid split.

## Description

Three queries are exposed here:

- **`AccountBalanceQuery`** extends `Query<AccountBalance>` — a free read that returns the
  native-token balance and per-token balances for an account or smart contract. The protocol
  does not charge for this query; no payment is built and the `PaidQuery` knobs do not apply.

- **`AccountInfoQuery`** extends `PaidQuery<AccountInfo>` — a paid read that returns the full
  state of an account (key, memo, expiration, staking configuration, EVM-address alias,
  etc.). Auto-discovered cost; the operator pays by default, or a `payer` `Account` may be
  set for sponsored reads.

- **`AccountRecordsQuery`** extends `PaidQuery<AccountRecords>` — a paid read that returns the
  recent transaction records in which the account was the effective payer. This is a legacy
  ("threshold records") surface: on modern Hiero / Hedera networks the list is effectively
  always empty (see *Questions & Comments*). Kept for V2 parity.

The set illustrates the free / paid split established by `consensusnode.queries`: a free
query's `submit(...)` returns a `QueryResponse<...>`, while a paid query's `submit(...)`
returns the richer `PaidQueryResponse<...>` with the actually-paid cost.

## API Schema

```
namespace consensusnode.queries.accounts
requires {Address, AccountId, ContractId, EvmAddress} from ledger
requires {PublicKey} from keys
requires {NativeToken} from nativeToken
requires {Query, PaidQuery} from consensusnode.queries
requires {Receipt, Record} from consensusnode.transactions

// Current balance snapshot of an account or contract. Returned by AccountBalanceQuery.
type AccountBalance {
    @@immutable accountId: AccountId                     // the account or contract this snapshot belongs to
    @@immutable balance: NativeToken<ANY, ANY>           // native-token balance
    @@immutable tokenBalances: map<Address, int64>       // tokenId -> balance in the token's smallest unit
}

// Free query for the current balance of an account or smart contract.
// Exactly one of accountId or contractId must be set.
@@finalType
@@oneOf(accountId, contractId)
AccountBalanceQuery extends Query<AccountBalance> {
    @@immutable @@nullable accountId: AccountId
    @@immutable @@nullable contractId: ContractId
}

// Full account state snapshot. Returned by AccountInfoQuery.
type AccountInfo {
    @@immutable accountId: AccountId
    @@immutable @@nullable evmAddress: EvmAddress                     // 20-byte EVM-address alias if assigned
    @@immutable balance: NativeToken<ANY, ANY>
    @@immutable @@nullable key: PublicKey
    @@immutable @@nullable accountMemo: string
    @@immutable expirationTime: zonedDateTime
    @@immutable @@nullable autoRenewPeriod: seconds
    @@immutable @@default(0) maxAutomaticTokenAssociations: int32
    @@immutable @@default(false) receiverSignatureRequired: bool
    @@immutable @@default(0) ownedNfts: int64
    @@immutable @@default(false) deleted: bool
    @@immutable @@nullable stakedAccountId: AccountId                 // mutually exclusive with stakedNodeId
    @@immutable @@nullable stakedNodeId: int64                        // mutually exclusive with stakedAccountId
    @@immutable @@default(false) declineStakingReward: bool
}

// Paid query for the full account state.
@@finalType
AccountInfoQuery extends PaidQuery<AccountInfo> {
    @@immutable accountId: AccountId
}

// The recent transaction records associated with an account. Returned by AccountRecordsQuery.
// `records` is heterogeneous — it mixes records of whatever transaction types the account
// paid for — so it is typed against the base `Record<Receipt>` rather than a concrete
// receipt subtype. The list is never null; an account with no qualifying records (the common
// case on modern networks) yields an empty list.
type AccountRecords {
    @@immutable accountId: AccountId
    @@immutable @@default([]) records: list<Record<Receipt>>
}

// Paid query for the recent transaction records in which `accountId` was the effective payer.
// Legacy "threshold records" surface — on modern Hiero / Hedera networks the returned list is
// effectively always empty (see Questions & Comments). Kept for V2 parity.
@@finalType
AccountRecordsQuery extends PaidQuery<AccountRecords> {
    @@immutable accountId: AccountId
}
```

## Examples

### Read a balance (free)

```
HieroClient client = ...;

AccountBalance balance = new AccountBalanceQuery()
    .accountId(...)
    .submit(client)
    .value;

NativeToken<ANY, ANY> nativeBalance = balance.balance;
int64 tokenAmount                    = balance.tokenBalances.get(someTokenId);
```

No cost discovery, no payment, no `maxQueryPayment`. `.value` unwraps the `QueryResponse`
envelope for callers that only need the payload.

### Read a balance with envelope metadata

```
QueryResponse<AccountBalance> response = new AccountBalanceQuery()
    .accountId(...)
    .submit(client);

AccountBalance balance = response.value;
AccountId answeringNode = response.answeredBy;  // useful for forensics
```

### Read full account state (paid)

```
PaidQueryResponse<AccountInfo> response = new AccountInfoQuery()
    .accountId(...)
    .maxQueryPayment(NativeToken.of(...))       // bound the spend
    .submit(client);

AccountInfo info = response.value;
PublicKey   key  = info.key;
NativeToken<ANY, ANY> paid = response.cost;     // what was actually charged
```

The client's operator pays. A configurable payer (for paymaster / custodial / sponsored
patterns) is deferred — see
[ADR-0002](../../docs/adr/0002-defer-paid-query-payer-customization.md).

### Read an account's recent records (paid)

```
AccountRecords result = new AccountRecordsQuery()
    .accountId(...)
    .submit(client)
    .value;

for (Record<Receipt> record : result.records) {
    TransactionId id = record.transactionId;
    TransactionStatus status = record.receipt.status;
}
```

`records` is typed against the base `Record<Receipt>`; callers that know a specific entry's
transaction type can narrow its `receipt` at the language level. On modern networks the list
is typically empty — see *Questions & Comments*.

## Questions & Comments

- **`AccountBalanceQuery` accepts either `accountId: AccountId` or `contractId: ContractId`.**
  Both target the same underlying entity space at the protocol level, but they are distinct
  typed identifiers in V3 (see [ADR-0003](../../docs/adr/0003-three-level-address-hierarchy-with-nullability-narrowing.md)).
  The dual setter matches the V2 ergonomic of having a balance query work uniformly for
  accounts and smart contracts. `@@oneOf` enforces exactly one of the two at construction.
- **`AccountInfo.evmAddress` is the typed `EvmAddress` value type.** Carries 20 raw bytes
  with hex `toString()` and `fromString` / `fromBytes` factories — concrete language bindings
  do not need to expose a separate hex-string accessor.
- **Deleted-account behaviour.** Querying a deleted account returns an `AccountInfo` with
  `deleted = true` and most other fields cleared. Callers should branch on `deleted` rather
  than assuming the rest of the snapshot is meaningful.
- **`AccountInfo` here is distinct from `mirrornode.account.AccountInfo`.** Same conceptual
  data but different sources (consensus node gRPC vs. mirror node REST) and slightly
  different field semantics (e.g. consensus-node `balance` is always live; mirror-node
  `balance` carries a `balanceTimestamp`). The two are kept separate to avoid coupling the
  consensus-node API to mirror-node update cadence.

- **`AccountRecordsQuery` is a legacy surface and almost always returns an empty list.** HAPI's
  `CryptoGetAccountRecords` returns *threshold records* — records the network retained for an
  account because a transaction's transfer crossed the (now removed)
  `sendRecordThreshold` / `receiveRecordThreshold` configured on the account. Those threshold
  fields were deprecated and the records are no longer generated on current Hiero / Hedera
  networks, so the list is effectively always empty. The query is kept for V2 parity; callers
  that want an account's transaction history should use the mirror node
  (`mirrornode.transaction`) instead, which is the supported path and is free. We may drop this
  query before V3 GA if no consumer needs the parity — flagged for review.

- **`AccountRecords.records` is typed `list<Record<Receipt>>`, not a concrete record type.**
  The list mixes records of whatever transaction types the account paid for, so it is bound to
  the base abstraction `Record<Receipt>` from [`transactions.md`](transactions.md). Callers that
  need a typed receipt narrow the element at the language level (e.g. a pattern match / `instanceof`).
  This is the only place in the query specs that returns the base `Record` rather than a
  query-specific result type.
