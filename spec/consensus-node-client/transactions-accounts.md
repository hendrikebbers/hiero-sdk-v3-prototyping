# Account Transactions API


## Description

## API Schema

```
namespace consensusnode.transactions.accounts
requires {Address} from ledger
requires {PublicKey} from keys
requires {NativeToken} from nativeToken
requires {Receipt, Transaction} from consensusnode.transactions

@@finalType
AccountCreateTransaction extends Transaction<AccountCreateReceipt> {
    @@immutable key: PublicKey
    @@immutable @@default(0) initialBalance: NativeToken<ANY, ANY>
    @@immutable @@nullable accountMemo: string
    @@immutable @@default(false) receiverSignatureRequired: bool
    @@immutable @@nullable maxAutomaticTokenAssociations: int32
    @@immutable @@nullable autoRenewPeriod: int64
    @@immutable @@nullable stakedAccountId: Address
    @@immutable @@nullable stakedNodeId: int64
    @@immutable @@default(false) declineStakingReward: bool
    @@immutable @@nullable alias: bytes
}

@@finalType
AccountCreateReceipt extends Receipt {
    @@immutable accountId: Address
}

@@finalType
AccountUpdateTransaction extends Transaction<AccountUpdateReceipt> {
    @@immutable accountId: Address                              // the account that is being updated
    @@immutable @@nullable key: PublicKey                          // the new key (requires signatures with both old and new keys)
    @@immutable @@nullable accountMemo: string
    @@immutable @@nullable receiverSignatureRequired: bool
    @@immutable @@nullable maxAutomaticTokenAssociations: int32
    @@immutable @@nullable autoRenewPeriod: int64
    @@immutable @@nullable expirationTime: zonedDateTime
    @@immutable @@nullable stakedAccountId: Address              // mutually exclusive with stakedNodeId
    @@immutable @@nullable stakedNodeId: int64                          // mutually exclusive with stakedAccountId
    @@immutable @@nullable declineStakingReward: bool
}

@@finalType
AccountUpdateReceipt extends Receipt {
}

@@finalType
AccountDeleteTransaction extends Transaction<AccountDeleteReceipt> {
    @@immutable accountId: Address                              // the account that is being deleted (must sign)
    @@immutable transferAccountId: Address                       // the account that receives the remaining hbar balance
}

@@finalType
AccountDeleteReceipt extends Receipt {
}
```

## Examples

## Questions & Comments
