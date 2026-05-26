# Mirror Node Common API

## Description

## API Schema

```
namespace mirrornode.common
requires {Address} from ledger
requires {Page} from common

@@finalType
Transfer {
    @@immutable account: Address
    @@immutable amount: int64
    @@immutable isApproval: bool
}

@@finalType
TokenTransfer {
    @@immutable tokenId: Address
    @@immutable account: Address
    @@immutable amount: int64
    @@immutable isApproval: bool
}

// Grouping of sender, receiver, and token for an NFT transfer
@@finalType
NftTransferParties {
    @@immutable senderAccountId: Address
    @@immutable receiverAccountId: Address
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
    @@immutable account: Address
    @@immutable amount: int64
}

FixedFee {
    @@immutable amount: int64
    @@immutable @@nullable collectorAccountId: Address
    @@immutable @@nullable denominatingTokenId: Address //TODO: Does this makes sense since it is used in topic query service
}
```
