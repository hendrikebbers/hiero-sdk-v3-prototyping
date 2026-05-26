# Mirror Node Account Query API

## Description

## API Schema

```
namespace mirrornode.account
requires {Address, MirrorNode} from ledger
requires {PublicKey} from keys
requires {Page} from common

@@finalType
AccountInfo {
    @@immutable accountId: Address
    @@immutable @@nullable evmAddress: string
    @@immutable balance: int64                                       // hbar balance in tinybars
    @@immutable @@nullable balanceTimestamp: zonedDateTime           // timestamp of the reported balance
    @@immutable ethereumNonce: int64
    @@immutable pendingReward: int64                                 // tinybars receivable in the next staking payout
    @@immutable @@nullable alias: string                             // RFC4648 base32-encoded account alias
    @@immutable @@nullable key: PublicKey
    @@immutable @@nullable accountMemo: string
    @@immutable @@nullable createdTimestamp: zonedDateTime
    @@immutable @@nullable expiryTimestamp: zonedDateTime
    @@immutable @@nullable autoRenewPeriod: int64
    @@immutable @@nullable maxAutomaticTokenAssociations: int32
    @@immutable @@nullable receiverSignatureRequired: bool
    @@immutable @@default(false) declineReward: bool
    @@immutable @@default(false) deleted: bool
    @@immutable @@nullable stakedAccountId: Address           // mutually exclusive with stakedNodeId
    @@immutable @@nullable stakedNodeId: int64                       // mutually exclusive with stakedAccountId
    @@immutable @@nullable stakePeriodStart: zonedDateTime
}

abstraction AccountRepository {
    @@async @@throws(mirror-node-error)
    @@nullable AccountInfo findById(accountId: Address)

    // Lists account entities known to the mirror node.
    // Maps to GET /api/v1/accounts.
    @@async @@throws(mirror-node-error)
    Page<AccountInfo> findAll()
}

@static createRepository(mirrorNode: MirrorNode)

```
