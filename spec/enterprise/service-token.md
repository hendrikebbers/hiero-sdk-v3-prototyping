# Fungible Token Service API

Service definition for creating and managing fungible tokens (the Token Service).

## Description

Provides high-level operations for fungible tokens: creation, account association, minting, burning, and transfers.
Token and account ids are `ledger.Address` values; amounts are expressed in the token's smallest unit as `int64`.
A token that should support later minting/burning must be created with a supply key.

## API Schema

```
namespace enterprise.service.token
requires {Page} from common
requires {Address} from ledger
requires {PublicKey} from keys
requires {Token, TokenInfo, Balance} from mirrornode.token
requires {Session} from enterprise.service

FungibleTokenService {

    // Create a fungible token. The treasury defaults to the operator account; a supply key enables mint/burn.
    @@throws(service-error) Address createToken(name: string, symbol: string)

    @@throws(service-error) Address createToken(name: string, symbol: string, supplyKey: PublicKey)

    @@throws(service-error) Address createToken(name: string, symbol: string, treasuryAccount: Address, supplyKey: PublicKey)

    // Associate an account with one or more tokens so it can hold them
    @@throws(service-error) void associateToken(accountId: Address, tokenIds: Address...)

    // Remove the association between an account and one or more tokens
    @@throws(service-error) void dissociateToken(accountId: Address, tokenIds: Address...)

    // Mint new units into the treasury; returns the new total supply
    @@throws(service-error) int64 mintToken(tokenId: Address, amount: int64)

    // Burn units from the treasury; returns the new total supply
    @@throws(service-error) int64 burnToken(tokenId: Address, amount: int64)

    // Transfer units from the operator account to a recipient
    @@throws(service-error) void transferToken(tokenId: Address, toAccountId: Address, amount: int64)

    // Transfer units from a specific account to a recipient
    @@throws(service-error) void transferToken(tokenId: Address, fromAccountId: Address, toAccountId: Address, amount: int64)

    // Return the full token information for the given token id
    @@throws(service-error) @@nullable TokenInfo findById(tokenId: Address)

    // Return all tokens that the given account is associated with
    @@throws(service-error) Page<Token> findByAccount(accountId: Address)

    // Return the balance of the given token held by every account that holds it
    @@throws(service-error) Page<Balance> getBalances(tokenId: Address)

    // Return the balance of the given token for a specific account
    @@throws(service-error) Page<Balance> getBalancesForAccount(tokenId: Address, accountId: Address)
}

// Factory method to create the service (not needed for real framework integration where injection is used)
@@static
FungibleTokenService createService(session: Session)
```

## Questions & Comments
