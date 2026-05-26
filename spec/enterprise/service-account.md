# Account Service API

Service definition for account handling.

## API Schema

```
namespace enterprise.service.account
requires common, ledger, nativeToken, consensusnode.client

AccountInformation {
    @@immutable accountId: ledger.Address
    @@immutable balance: nativeToken.NativeToken<ANY, ANY>
    @@immutable key: keys.PublicKey
}

AccountService {

  @@throws(service-error) ledger.Address createNewAccount(key: keys.PublicKey)
  
  @@throws(service-error) ledger.Address createNewAccount(key: keys.PublicKey, initialBalance: nativeToken.NativeToken<ANY, ANY>)

  @@throws(service-error) void updateAccountKey(accountId: ledger.Address, newKey: keys.PublicKey, oldKey: keys.PublicKey)
  
  @@throws(service-error) void deleteAccount(account: ledger.Address)

  @@throws(service-error) AccountInformation findById(account: ledger.Address)
  
  @@throws(service-error) common.Page<AccountInformation> findAll()
}

//Factory method to create Service (not needed for real framework integration where injection is used)
@@static
AccountService createService(networkSettings: ledger.config.NetworkSetting, operatorAccount: consensusnode.client.Account)

@@static
AccountService createService(networkSettings: ledger.config.NetworkSetting, operatorAccount: consensusnode.client.Account, transactionSigner: consensusnode.client.TransactionSigner)

```