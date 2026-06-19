# Mirror Node Token Query API

## Description

## API Schema

```
namespace mirrornode.token
requires {Address, AccountId, MirrorNode} from ledger
requires {FixedFee} from mirrornode.common
requires {Page} from common
requires {TokenType, TokenSupplyType} from token

@@finalType
Balance {
    @@immutable accountId: AccountId
    @@immutable balance: int64
    @@immutable decimals: int64
}

@@finalType
RoyaltyFee {
    @@immutable numeratorAmount: int64
    @@immutable denominatorAmount: int64
    @@immutable fallbackFeeAmount: int64
    @@immutable @@nullable collectorAccountId: AccountId
    @@immutable @@nullable denominatingTokenId: Address
}

@@finalType
FractionalFee {
    @@immutable numeratorAmount: int64
    @@immutable denominatorAmount: int64
    @@immutable @@nullable collectorAccountId: AccountId
    @@immutable @@nullable denominatingTokenId: Address
}


@@finalType
CustomFee {
    @@immutable fixedFees: list<FixedFee>
    @@immutable fractionalFees: list<FractionalFee>
    @@immutable royaltyFees: list<RoyaltyFee>
}

@@finalType
TokenInfo {
    @@immutable tokenId: Address
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
    @@immutable treasuryAccountId: AccountId
    @@immutable deleted: bool
    @@immutable customFees: CustomFee
}

@@finalType
Token {
    @@immutable @@nullable tokenId: Address
    @@immutable name: string
    @@immutable symbol: string
    @@immutable type: TokenType
    @@immutable decimals: int64
    @@immutable metadata: bytes
}


abstraction TokenRepository {
    @@async @@throws(mirror-node-error)
    Page<Token> findByAccount(accountId: AccountId)

    @@async @@throws(mirror-node-error)
    @@nullable TokenInfo findById(tokenId: Address)

    @@async @@throws(mirror-node-error)
    Page<Balance> getBalances(tokenId: Address)

    @@async @@throws(mirror-node-error)
    Page<Balance> getBalancesForAccount(tokenId: Address, accountId: AccountId)
}

@@static TokenRepository createRepository(mirrorNode: MirrorNode)

```
