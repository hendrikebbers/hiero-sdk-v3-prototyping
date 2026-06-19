# Mirror Node Common API

## Description

## API Schema

```
namespace mirrornode.common
requires {Address, AccountId} from ledger

@@finalType
Transfer {
    @@immutable account: AccountId
    @@immutable amount: int64
    @@immutable isApproval: bool
}

@@finalType
TokenTransfer {
    @@immutable tokenId: Address
    @@immutable account: AccountId
    @@immutable amount: int64
    @@immutable isApproval: bool
}

// Grouping of sender, receiver, and token for an NFT transfer
@@finalType
NftTransferParties {
    @@immutable senderAccountId: AccountId
    @@immutable receiverAccountId: AccountId
    @@immutable tokenId: Address
}

@@finalType
NftTransfer {
    @@immutable isApproval: bool
    @@immutable @@nullable parties: NftTransferParties
    @@immutable serialNumber: int64
}

@@finalType
StakingRewardTransfer {
    @@immutable account: AccountId
    @@immutable amount: int64
}

FixedFee {
    @@immutable amount: int64
    @@immutable @@nullable collectorAccountId: AccountId
    @@immutable @@nullable denominatingTokenId: Address //TODO: Does this makes sense since it is used in topic query service
}
```
