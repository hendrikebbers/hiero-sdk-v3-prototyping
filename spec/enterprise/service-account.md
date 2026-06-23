# Account Service API

Service definition for account handling.

## API Schema

```
namespace enterprise.service.account
requires {Page} from common
requires {AccountId} from ledger
requires {Authority} from authority
requires {NativeToken} from nativeToken
requires {Session} from enterprise.service

AccountInformation {
    @@immutable accountId: AccountId
    @@immutable balance: NativeToken<ANY, ANY>
    @@immutable authority: Authority
}

AccountService {

  @@throws(service-error) AccountId createNewAccount(authority: Authority)

  @@throws(service-error) AccountId createNewAccount(authority: Authority, initialBalance: NativeToken<ANY, ANY>)

  @@throws(service-error) void updateAccountAuthority(accountId: AccountId, newAuthority: Authority, oldAuthority: Authority)

  @@throws(service-error) void deleteAccount(account: AccountId)

  @@throws(service-error) AccountInformation findById(account: AccountId)

  @@throws(service-error) Page<AccountInformation> findAll()
}

//Factory method to create Service (not needed for real framework integration where injection is used)
@@static
AccountService createService(session: Session)

```

## Questions & Comments

- The authorization parameters/fields here (`AccountInformation.authority`, `createNewAccount(authority)`, and both
  `newAuthority`/`oldAuthority` of `updateAccountAuthority`) accept the full `Authority` type, so multisig (m-of-n) and
  contract-controlled keys work at the enterprise layer too — not just single public keys. See
  [ADR-0004](../../docs/adr/0004-authority-authorization-sum-type.md) and [`authority.md`](../base/authority.md).
- Open option: simple single-key convenience overloads taking a `PublicKey` directly could be added later if the
  enterprise layer wants extra ergonomics for the common single-signer case.