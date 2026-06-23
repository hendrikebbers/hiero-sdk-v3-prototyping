# Mirror Node Account Query API

## Description

Read-side queries for accounts: the account entity (`findById` / `findAll`), staking-reward
payouts, the HBAR / fungible-token / NFT allowances an account has granted, HIP-904 pending
airdrops (both sent-and-outstanding and to-be-received), and the network-wide balances snapshot.

## API Schema

```
namespace mirrornode.account
requires {Address, AccountId, EvmAddress, MirrorNode} from ledger
requires {Authority} from authority
requires {Page} from common

@@finalType
AccountInfo {
    @@immutable accountId: AccountId
    @@immutable @@nullable evmAddress: EvmAddress
    @@immutable balance: int64                                       // hbar balance in tinybars
    @@immutable @@nullable balanceTimestamp: zonedDateTime           // timestamp of the reported balance
    @@immutable ethereumNonce: int64
    @@immutable pendingReward: int64                                 // tinybars receivable in the next staking payout
    @@immutable @@nullable alias: bytes                              // HIP-32 public-key alias (serialised protobuf Key bytes)
    @@immutable @@nullable authority: Authority
    @@immutable @@nullable accountMemo: string
    @@immutable @@nullable createdTimestamp: zonedDateTime
    @@immutable @@nullable expiryTimestamp: zonedDateTime
    @@immutable @@nullable autoRenewPeriod: seconds
    @@immutable @@nullable maxAutomaticTokenAssociations: int32
    @@immutable @@nullable receiverSignatureRequired: bool
    @@immutable @@default(false) declineReward: bool
    @@immutable @@default(false) deleted: bool
    @@immutable @@nullable stakedAccountId: AccountId                // mutually exclusive with stakedNodeId
    @@immutable @@nullable stakedNodeId: int64                       // mutually exclusive with stakedAccountId
    @@immutable @@nullable stakePeriodStart: zonedDateTime
}

// A staking-reward payout received by the account. GET /accounts/{id}/rewards.
@@finalType
StakingReward {
    @@immutable accountId: AccountId
    @@immutable amount: int64                         // tinybars paid in this reward
    @@immutable timestamp: zonedDateTime              // consensus time of the payout
}

// An HBAR spending allowance the account granted. GET /accounts/{id}/allowances/crypto.
@@finalType
CryptoAllowance {
    @@immutable owner: AccountId
    @@immutable spender: AccountId
    @@immutable amount: int64                         // remaining approved tinybars
    @@immutable amountGranted: int64                  // originally approved tinybars
    @@immutable timestamp: zonedDateTime              // when the allowance was granted / last modified
}

// A fungible-token spending allowance the account granted. GET /accounts/{id}/allowances/tokens.
@@finalType
TokenAllowance {
    @@immutable owner: AccountId
    @@immutable spender: AccountId
    @@immutable tokenId: Address
    @@immutable amount: int64                         // remaining approved amount (smallest unit)
    @@immutable amountGranted: int64                  // originally approved amount
    @@immutable timestamp: zonedDateTime
}

// An NFT approved-for-all allowance the account granted. GET /accounts/{id}/allowances/nfts.
@@finalType
NftAllowance {
    @@immutable owner: AccountId
    @@immutable spender: AccountId
    @@immutable tokenId: Address
    @@immutable approvedForAll: bool                  // true → spender may transfer every serial of the collection
    @@immutable timestamp: zonedDateTime
}

// A pending airdrop (HIP-904) the account sent (outstanding) or is to receive (pending).
// Exactly one of `amount` (fungible) / `serialNumber` (NFT) is set.
@@finalType
TokenAirdrop {
    @@immutable senderId: AccountId
    @@immutable receiverId: AccountId
    @@immutable tokenId: Address
    @@immutable @@nullable amount: int64              // fungible amount (smallest unit); null for an NFT airdrop
    @@immutable @@nullable serialNumber: int64        // NFT serial; null for a fungible airdrop
    @@immutable timestamp: zonedDateTime
}

// One token balance line inside an AccountBalance.
@@finalType
TokenBalance {
    @@immutable tokenId: Address
    @@immutable balance: int64                        // balance in the token's smallest unit
}

// An account's balance snapshot from the top-level balances endpoint. GET /balances.
@@finalType
AccountBalance {
    @@immutable account: AccountId
    @@immutable balance: int64                        // hbar balance in tinybars
    @@immutable @@default([]) tokens: list<TokenBalance>
}

abstraction AccountRepository {
    @@async @@throws(mirror-node-error)
    @@nullable AccountInfo findById(accountId: AccountId)

    // Lists account entities known to the mirror node.
    // Maps to GET /api/v1/accounts.
    @@async @@throws(mirror-node-error)
    Page<AccountInfo> findAll()

    // Staking-reward payouts received by the account.
    // Maps to GET /api/v1/accounts/{id}/rewards.
    @@async @@throws(mirror-node-error)
    Page<StakingReward> getRewards(accountId: AccountId)

    // HBAR allowances the account has granted to spenders.
    // Maps to GET /api/v1/accounts/{id}/allowances/crypto.
    @@async @@throws(mirror-node-error)
    Page<CryptoAllowance> getCryptoAllowances(accountId: AccountId)

    // Fungible-token allowances the account has granted.
    // Maps to GET /api/v1/accounts/{id}/allowances/tokens.
    @@async @@throws(mirror-node-error)
    Page<TokenAllowance> getTokenAllowances(accountId: AccountId)

    // NFT approved-for-all allowances the account has granted.
    // Maps to GET /api/v1/accounts/{id}/allowances/nfts.
    @@async @@throws(mirror-node-error)
    Page<NftAllowance> getNftAllowances(accountId: AccountId)

    // Pending airdrops the account has SENT that are not yet claimed (HIP-904).
    // Maps to GET /api/v1/accounts/{id}/airdrops/outstanding.
    @@async @@throws(mirror-node-error)
    Page<TokenAirdrop> getOutstandingAirdrops(accountId: AccountId)

    // Pending airdrops the account is to RECEIVE but has not yet claimed (HIP-904).
    // Maps to GET /api/v1/accounts/{id}/airdrops/pending.
    @@async @@throws(mirror-node-error)
    Page<TokenAirdrop> getPendingAirdrops(accountId: AccountId)

    // Account balances across the network (latest, or at a queried timestamp).
    // Maps to the top-level GET /api/v1/balances — NOT /accounts/{id}/balances.
    @@async @@throws(mirror-node-error)
    Page<AccountBalance> getBalances()
}

@@static AccountRepository createRepository(mirrorNode: MirrorNode)

```

## Questions & Comments

- **Allowance `timestamp` is a single value, not a range.** The Mirror Node REST returns a
  `timestamp` *range* (from/to) on allowances; V3 surfaces the grant / last-modified instant as a
  single `zonedDateTime`, consistent with the other mirror-node types. A typed `TimestampRange`
  (for the generic range-filter model) is tracked in [`missing-features.md`](../../missing-features.md) §4.3.
- **`getNftAllowances` returns approved-for-all grants only.** `GET /allowances/nfts` exposes
  collection-wide (`approvedForAll`) NFT allowances; per-serial NFT approvals are read via the NFT
  endpoints, not here.
- **`TokenAirdrop` is one shape for both directions.** *Outstanding* = sent by this account and not
  yet claimed; *pending* = receivable by this account. `amount` (fungible) and `serialNumber` (NFT)
  are mutually exclusive.
- **`getBalances()` maps to the top-level `/balances`**, not a per-account path — it returns
  balances across accounts. It is placed on `AccountRepository` because the result is
  account-balance-shaped, and is distinct from `mirrornode.token.Balance` (which is token-centric).
