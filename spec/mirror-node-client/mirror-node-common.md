# Mirror Node Common API

## Description

## API Schema

```
namespace mirrornode.common
requires ledger, keys

@@finalType
Transfer {
    @@immutable account: ledger.Address
    @@immutable amount: int64
    @@immutable isApproval: bool
}

@@finalType
TokenTransfer {
    @@immutable tokenId: ledger.Address
    @@immutable account: ledger.Address
    @@immutable amount: int64
    @@immutable isApproval: bool
}

// Grouping of sender, receiver, and token for an NFT transfer
@@finalType
NftTransferParties {
    @@immutable senderAccountId: ledger.Address
    @@immutable receiverAccountId: ledger.Address
    @@immutable tokenId: ledger.Address
}

@@finalType
NftTransfer {
    @@immutable isApproval: bool
    @@immutable @@nullable parties: NftTransferParties
    @@immutable serialNumber: int64
}

@@finalType
StakingRewardTransfer {
    @@immutable account: ledger.Address
    @@immutable amount: int64
}

FixedFee {
    @@immutable amount: int64
    @@immutable @@nullable collectorAccountId: ledger.Address
    @@immutable @@nullable denominatingTokenId: ledger.Address //TODO: Does this makes sense since it is used in topic query service
}
```
