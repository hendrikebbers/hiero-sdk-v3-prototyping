# Account Service API

Service definition for account handling.

## API Schema

```
namespace enterprise.service.account
requires {Page} from common
requires {AccountId} from ledger
requires {PublicKey} from keys
requires {NativeToken} from nativeToken
requires {Session} from enterprise.service

AccountInformation {
    @@immutable accountId: AccountId
    @@immutable balance: NativeToken<ANY, ANY>
    @@immutable key: PublicKey
}

AccountService {

  @@throws(service-error) AccountId createNewAccount(key: PublicKey)

  @@throws(service-error) AccountId createNewAccount(key: PublicKey, initialBalance: NativeToken<ANY, ANY>)

  @@throws(service-error) void updateAccountKey(accountId: AccountId, newKey: PublicKey, oldKey: PublicKey)

  @@throws(service-error) void deleteAccount(account: AccountId)

  @@throws(service-error) AccountInformation findById(account: AccountId)

  @@throws(service-error) Page<AccountInformation> findAll()
}

//Factory method to create Service (not needed for real framework integration where injection is used)
@@static
AccountService createService(session: Session)

```