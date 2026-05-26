# NFT Service API

Service definition for creating and managing non-fungible tokens (NFTs).

## Description

Provides high-level operations for NFTs: creating an NFT type (collection), account association, minting NFTs with
metadata, burning, and transfers. An NFT type and the accounts are `ledger.Address` values; individual NFTs are
identified by their type plus an `int64` serial number. Metadata is an opaque `bytes` payload.

## API Schema

```
namespace enterprise.service.nft
requires {Address} from ledger
requires {NetworkSetting} from ledger.config
requires {PublicKey} from keys
requires {Account, TransactionSigner} from consensusnode.client

NftService {

    // Create a new NFT type (collection); returns the token id of the new type.
    // The treasury defaults to the operator account; a supply key enables minting.
    @@throws(service-error) Address createNftType(name: string, symbol: string)

    @@throws(service-error) Address createNftType(name: string, symbol: string, supplyKey: PublicKey)

    @@throws(service-error) Address createNftType(name: string, symbol: string, treasuryAccount: Address, supplyKey: PublicKey)

    // Associate an account with one or more NFT types so it can hold them
    @@throws(service-error) void associateNft(accountId: Address, tokenIds: Address...)

    // Remove the association between an account and one or more NFT types
    @@throws(service-error) void dissociateNft(accountId: Address, tokenIds: Address...)

    // Mint a single NFT with the given metadata; returns the new serial number
    @@throws(service-error) int64 mintNft(tokenId: Address, metadata: bytes)

    // Mint multiple NFTs in one operation; returns the new serial numbers in order
    @@throws(service-error) list<int64> mintNfts(tokenId: Address, metadata: bytes...)

    // Burn a single NFT by serial number
    @@throws(service-error) void burnNft(tokenId: Address, serialNumber: int64)

    // Burn multiple NFTs by serial number
    @@throws(service-error) void burnNfts(tokenId: Address, serialNumbers: set<int64>)

    // Transfer a single NFT to another account
    @@throws(service-error) void transferNft(tokenId: Address, serialNumber: int64, fromAccountId: Address, toAccountId: Address)

    // Transfer multiple NFTs of the same type to another account
    @@throws(service-error) void transferNfts(tokenId: Address, serialNumbers: list<int64>, fromAccountId: Address, toAccountId: Address)
}

// Factory methods to create the service (not needed for real framework integration where injection is used)
@@static
NftService createService(networkSettings: NetworkSetting, operatorAccount: Account)

@@static
NftService createService(networkSettings: NetworkSetting, operatorAccount: Account, transactionSigner: TransactionSigner)
```

## Questions & Comments
