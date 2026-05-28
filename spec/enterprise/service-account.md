# Account Service API

Service definition for account handling.

## API Schema

```
namespace enterprise.service.account
requires {Page} from common
requires {Address} from ledger
requires {PublicKey} from keys
requires {NativeToken} from nativeToken
requires {Session} from enterprise.service

AccountInformation {
    @@immutable accountId: Address
    @@immutable balance: NativeToken<ANY, ANY>
    @@immutable key: PublicKey
}

AccountService {

  @@throws(service-error) Address createNewAccount(key: PublicKey)
  
  @@throws(service-error) Address createNewAccount(key: PublicKey, initialBalance: NativeToken<ANY, ANY>)

  @@throws(service-error) void updateAccountKey(accountId: Address, newKey: PublicKey, oldKey: PublicKey)
  
  @@throws(service-error) void deleteAccount(account: Address)

  @@throws(service-error) AccountInformation findById(account: Address)
  
  @@throws(service-error) Page<AccountInformation> findAll()
}

//Factory method to create Service (not needed for real framework integration where injection is used)
@@static
AccountService createService(session: Session)

```