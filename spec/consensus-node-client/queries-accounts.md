# Account Queries API

This namespace defines account-related queries against the consensus node. It is the first
concrete usage of the abstractions defined in [`queries.md`](queries.md) and serves as a
worked example of the free / paid split.

## Description

Two queries are exposed here, one of each kind:

- **`AccountBalanceQuery`** extends `Query<AccountBalance>` — a free read that returns the
  native-token balance and per-token balances for an account or smart contract. The protocol
  does not charge for this query; no payment is built and the `PaidQuery` knobs do not apply.

- **`AccountInfoQuery`** extends `PaidQuery<AccountInfo>` — a paid read that returns the full
  state of an account (key, memo, expiration, staking configuration, EVM-address alias,
  etc.). Auto-discovered cost; the operator pays by default, or a `payer` `Account` may be
  set for sponsored reads.

The pair illustrates the free / paid split established by `consensusnode.queries`: a free
query's `submit(...)` returns a `QueryResponse<...>`, while a paid query's `submit(...)`
returns the richer `PaidQueryResponse<...>` with the actually-paid cost.

## API Schema

```
namespace consensusnode.queries.accounts
requires {Address} from ledger
requires {PublicKey} from keys
requires {NativeToken} from nativeToken
requires {Query, PaidQuery} from consensusnode.queries

// Current balance snapshot of an account or contract. Returned by AccountBalanceQuery.
type AccountBalance {
    @@immutable accountId: Address                       // the account or contract this snapshot belongs to
    @@immutable balance: NativeToken<ANY, ANY>           // native-token balance
    @@immutable tokenBalances: map<Address, int64>       // tokenId -> balance in the token's smallest unit
}

// Free query for the current balance of an account or smart contract.
// Exactly one of accountId or contractId must be set.
@@finalType
@@oneOf(accountId, contractId)
AccountBalanceQuery extends Query<AccountBalance> {
    @@immutable @@nullable accountId: Address
    @@immutable @@nullable contractId: Address
}

// Full account state snapshot. Returned by AccountInfoQuery.
type AccountInfo {
    @@immutable accountId: Address
    @@immutable @@nullable evmAddress: bytes                          // 20-byte EVM-address alias if assigned
    @@immutable balance: NativeToken<ANY, ANY>
    @@immutable @@nullable key: PublicKey
    @@immutable @@nullable accountMemo: string
    @@immutable expirationTime: zonedDateTime
    @@immutable @@nullable autoRenewPeriod: seconds
    @@immutable @@default(0) maxAutomaticTokenAssociations: int32
    @@immutable @@default(false) receiverSignatureRequired: bool
    @@immutable @@default(0) ownedNfts: int64
    @@immutable @@default(false) deleted: bool
    @@immutable @@nullable stakedAccountId: Address                   // mutually exclusive with stakedNodeId
    @@immutable @@nullable stakedNodeId: int64                        // mutually exclusive with stakedAccountId
    @@immutable @@default(false) declineStakingReward: bool
}

// Paid query for the full account state.
@@finalType
AccountInfoQuery extends PaidQuery<AccountInfo> {
    @@immutable accountId: Address
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
Address answeringNode  = response.answeredBy;   // useful for forensics
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

## Questions & Comments

- **`AccountBalanceQuery` accepts either `accountId` or `contractId`.** Both target the same
  underlying entity space at the protocol level; the dual setter matches the V2 ergonomic of
  having a balance query work uniformly for accounts and smart contracts. `@@oneOf` enforces
  exactly one of the two at construction.
- **Open: collapse `accountId` / `contractId` into a single field.** V2's dual setter exists
  because protobuf distinguishes the `AccountID` and `ContractID` message types in its
  `oneof balanceSource`, and V2 surfaced both for caller ergonomics. In V3 both collapse to
  `Address`, and on Hedera every contract owns an account at the same `shard.realm.num` —
  the consensus node returns the same `AccountBalance` regardless of which `oneof` branch
  was filled. A single `accountId: Address` field would therefore be sufficient (the SDK
  always fills the `accountID` proto branch internally, which works for contract addresses
  too). Trade-off: would drop the `@@oneOf`-enforced disambiguation and the explicit
  `setContractId(...)` ergonomic, in exchange for a smaller, type-honest surface. To be
  decided.
- **`AccountInfo.evmAddress` is `bytes`, not a hex string.** Concrete language bindings
  should expose it as a 20-byte value. Hex serialisation belongs at the rendering layer, not
  in the data model.
- **Deleted-account behaviour.** Querying a deleted account returns an `AccountInfo` with
  `deleted = true` and most other fields cleared. Callers should branch on `deleted` rather
  than assuming the rest of the snapshot is meaningful.
- **`AccountInfo` here is distinct from `mirrornode.account.AccountInfo`.** Same conceptual
  data but different sources (consensus node gRPC vs. mirror node REST) and slightly
  different field semantics (e.g. consensus-node `balance` is always live; mirror-node
  `balance` carries a `balanceTimestamp`). The two are kept separate to avoid coupling the
  consensus-node API to mirror-node update cadence.
