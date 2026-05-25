# Account Service API

Service definition for account handling.

## API Schema

```
namespace enterprise.service.account
requires ledger, hbar, consensusnode.client

AccountService {

  @@throws(service-error) ledger.Address createNewAccount(key: keys.PublicKey)
  
  @@throws(service-error) ledger.Address createNewAccount(key: keys.PublicKey, initialBalance: hbar.Hbar)

  @@throws(service-error) void updateAccountKey(accountId: ledger.Address, newKey: keys.PublicKey, oldKey: keys.PublicKey)
  
  @@throws(service-error) void deleteAccount(account: ledger.Address)

}

//Factory method to create Service (not needed for real framework integration where injection is used)
@@static
AccountService createService(networkSettings: ledger.config.NetworkSetting, operatorAccount: consensusnode.client.Account)

@@static
AccountService createService(networkSettings: ledger.config.NetworkSetting, operatorAccount: consensusnode.client.Account, transactionSigner: consensusnode.client.TransactionSigner)

```