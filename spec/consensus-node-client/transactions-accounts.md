# Account Transactions API


## Description

## API Schema

```
namespace consensus-node.transactions.transactionsAccounts
requires native-token, ledger, consensus-node.transactions, keys

@@finalType
AccountCreateTransaction extends consensus-node.transactions.Transaction<AccountCreateReceipt> {
    @@immutable key: keys.PublicKey
    @@immutable @@default(0) initialBalance: native-token.Hbar
    @@immutable @@nullable accountMemo: string
    @@immutable @@default(false) receiverSignatureRequired: bool
    @@immutable @@nullable maxAutomaticTokenAssociations: int32
    @@immutable @@nullable autoRenewPeriod: int64
    @@immutable @@nullable stakedAccountId: ledger.Address
    @@immutable @@nullable stakedNodeId: int64
    @@immutable @@default(false) declineStakingReward: bool
    @@immutable @@nullable alias: bytes
}

@@finalType
AccountCreateReceipt extends consensus-node.transactions.Receipt {
    @@immutable accountId: ledger.Address
}

@@finalType
AccountUpdateTransaction extends consensus-node.transactions.Transaction<AccountUpdateReceipt> {
    @@immutable accountId: ledger.Address                              // the account that is being updated
    @@immutable @@nullable key: keys.PublicKey                          // the new key (requires signatures with both old and new keys)
    @@immutable @@nullable accountMemo: string
    @@immutable @@nullable receiverSignatureRequired: bool
    @@immutable @@nullable maxAutomaticTokenAssociations: int32
    @@immutable @@nullable autoRenewPeriod: int64
    @@immutable @@nullable expirationTime: zonedDateTime
    @@immutable @@nullable stakedAccountId: ledger.Address              // mutually exclusive with stakedNodeId
    @@immutable @@nullable stakedNodeId: int64                          // mutually exclusive with stakedAccountId
    @@immutable @@nullable declineStakingReward: bool
}

@@finalType
AccountUpdateReceipt extends consensus-node.transactions.Receipt {
}

@@finalType
AccountDeleteTransaction extends consensus-node.transactions.Transaction<AccountDeleteReceipt> {
    @@immutable accountId: ledger.Address                              // the account that is being deleted (must sign)
    @@immutable transferAccountId: ledger.Address                       // the account that receives the remaining hbar balance
}

@@finalType
AccountDeleteReceipt extends consensus-node.transactions.Receipt {
}
```

## Examples

## Questions & Comments
