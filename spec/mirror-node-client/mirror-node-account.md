# Mirror Node Account Query API

## Description

## API Schema

```
namespace mirrornode.account
requires {AccountId, EvmAddress, MirrorNode} from ledger
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

abstraction AccountRepository {
    @@async @@throws(mirror-node-error)
    @@nullable AccountInfo findById(accountId: AccountId)

    // Lists account entities known to the mirror node.
    // Maps to GET /api/v1/accounts.
    @@async @@throws(mirror-node-error)
    Page<AccountInfo> findAll()
}

@@static AccountRepository createRepository(mirrorNode: MirrorNode)

```
