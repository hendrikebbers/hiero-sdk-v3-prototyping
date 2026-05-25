# Mirror Node Token Query API

## Description

## API Schema

```
namespace mirrornode.token
requires ledger, keys, mirrornode.common

@@finalType
Balance {
    @@immutable accountId: ledger.Address
    @@immutable balance: int64
    @@immutable decimals: int64
}

@@finalType
RoyaltyFee {
    @@immutable numeratorAmount: int64
    @@immutable denominatorAmount: int64
    @@immutable fallbackFeeAmount: int64
    @@immutable @@nullable collectorAccountId: ledger.Address
    @@immutable @@nullable denominatingTokenId: ledger.Address
}

@@finalType
FractionalFee {
    @@immutable numeratorAmount: int64
    @@immutable denominatorAmount: int64
    @@immutable @@nullable collectorAccountId: ledger.Address
    @@immutable @@nullable denominatingTokenId: ledger.Address
}


@@finalType
CustomFee {
    @@immutable fixedFees: list<FixedFee>
    @@immutable fractionalFees: list<FractionalFee>
    @@immutable royaltyFees: list<mirrornode.common.RoyaltyFee>
}

enum TokenSupplyType {
    INFINITE
    FINITE
}

enum TokenType {
    FUNGIBLE_COMMON
    NON_FUNGIBLE_UNIQUE
}


@@finalType
TokenInfo {
    @@immutable tokenId: ledger.Address
    @@immutable type: TokenType
    @@immutable name: string
    @@immutable symbol: string
    @@immutable @@nullable memo: string
    @@immutable decimals: int64
    @@immutable metadata: bytes
    @@immutable createdTimestamp: zonedDateTime
    @@immutable modifiedTimestamp: zonedDateTime
    @@immutable @@nullable expiryTimestamp: zonedDateTime
    @@immutable supplyType: TokenSupplyType
    @@immutable initialSupply: int256
    @@immutable totalSupply: int256
    @@immutable maxSupply: int256
    @@immutable treasuryAccountId: ledger.Address
    @@immutable deleted: bool
    @@immutable customFees: CustomFee
}

@@finalType
Token {
    @@immutable @@nullable tokenId: ledger.Address
    @@immutable name: string
    @@immutable symbol: string
    @@immutable type: TokenType
    @@immutable decimals: int64
    @@immutable metadata: bytes
}


abstraction TokenRepository {
    @@async @@throws(mirror-node-error)
    Page<Token> findByAccount(accountId: ledger.Address)

    @@async @@throws(mirror-node-error)
    @@nullable TokenInfo findById(tokenId: ledger.Address)

    @@async @@throws(mirror-node-error)
    Page<Balance> getBalances(tokenId: ledger.Address)

    @@async @@throws(mirror-node-error)
    Page<Balance> getBalancesForAccount(tokenId: ledger.Address, accountId: ledger.Address)
}

@static createRepository(mirrorNode: ledger.MirrorNode)

```
