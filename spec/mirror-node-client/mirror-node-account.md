# Mirror Node Account Query API

## Description

## API Schema

```
namespace mirrornode.account
requires ledger

@@finalType
AccountInfo {
    @@immutable accountId: ledger.Address
    @@immutable evmAddress: string
    @@immutable balance: int64
    @@immutable ethereumNonce: int64
    @@immutable pendingReward: int64
}

abstraction AccountRepository {
    @@async @@throws(mirror-node-error)
    @@nullable AccountInfo findById(accountId: ledger.Address)
}

@static createRepository(mirrorNode: ledger.MirrorNode)

```
