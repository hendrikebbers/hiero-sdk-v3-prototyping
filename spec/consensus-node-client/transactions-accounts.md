# Account Transactions API


## Description

## API Schema

```
namespace consensus-node.transactions.transactionsAccounts
requires native-token, ledger, consensus-node.transactions, keys

@@finalType
AccountCreateTransaction extends consensus-node.transactions.Transaction<AccountCreateReceipt> {
    @@immutable @@nullable accountMemo: string
    @@immutable @@default(0) initialBalance: native-token.Hbar
    @@immutable key: keys.PublicKey
}

@@finalType
AccountCreateReceipt extends consensus-node.transactions.Receipt {
    @@immutable accountId: ledger.Address
}
```

## Examples

## Questions & Comments
