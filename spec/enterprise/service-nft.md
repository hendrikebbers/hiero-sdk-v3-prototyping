# NFT Service API

Service definition for creating and managing non-fungible tokens (NFTs).

## Description

Provides high-level operations for NFTs: creating an NFT type (collection), account association, minting NFTs with
metadata, burning, and transfers. An NFT type and the accounts are `ledger.Address` values; individual NFTs are
identified by their type plus an `int64` serial number. Metadata is an opaque `bytes` payload.

## API Schema

```
namespace enterprise.service.nft
requires {Page} from common
requires {Address} from ledger
requires {PublicKey} from keys
requires {Nft, NftMetadata} from mirrornode.nft
requires {Session} from enterprise.service

NftService {

    // Create a new NFT type (collection); returns the token id of the new type.
    // The treasury defaults to the operator account; a supply key enables minting.
    @@throws(service-error) NftMetadata createNftType(name: string, symbol: string)

    @@throws(service-error) NftMetadata createNftType(name: string, symbol: string, supplyKey: PublicKey)

    @@throws(service-error) NftMetadata createNftType(name: string, symbol: string, treasuryAccount: Address, supplyKey: PublicKey)

    // Associate an account with one or more NFT types so it can hold them
    @@throws(service-error) void associateNft(accountId: Address, tokenIds: Address...)

    // Remove the association between an account and one or more NFT types
    @@throws(service-error) void dissociateNft(accountId: Address, tokenIds: Address...)

    // Mint a single NFT with the given metadata; returns the new serial number
    @@throws(service-error) Nft mintNft(tokenId: Address, metadata: bytes)

    // Mint multiple NFTs in one operation; returns the new serial numbers in order
    @@throws(service-error) list<Nft> mintNfts(tokenId: Address, metadata: bytes...)

    // Burn a single NFT by serial number
    @@throws(service-error) void burnNft(tokenId: Address, serialNumber: int64)
    
    // Burn a single NFT by serial number
    @@throws(service-error) void burnNft(nft: Nft)

    // Burn multiple NFTs by serial number
    @@throws(service-error) void burnNfts(tokenId: Address, serialNumbers: set<int64>)

    // Burn multiple NFTs by serial number
    @@throws(service-error) void burnNfts(nfts: set<Nft>)

    // Transfer a single NFT to another account
    @@throws(service-error) void transferNft(tokenId: Address, serialNumber: int64, fromAccountId: Address, toAccountId: Address)

    @@throws(service-error) void transferNfts(nft: Nft, fromAccountId: Address, toAccountId: Address)


    // Transfer multiple NFTs of the same type to another account
    @@throws(service-error) void transferNfts(tokenId: Address, serialNumbers: list<int64>, fromAccountId: Address, toAccountId: Address)

    @@throws(service-error) void transferNfts(nfts: set<Nft>, fromAccountId: Address, toAccountId: Address)

    // Return all known NFT types (collections)
    @@throws(service-error) Page<NftMetadata> findAllTypes()

    // Return all NFT types that the given account holds at least one NFT of
    @@throws(service-error) Page<NftMetadata> findTypesByOwner(ownerId: Address)

    // Return the metadata of a single NFT type
    @@throws(service-error) @@nullable NftMetadata findTypeById(tokenId: Address)

    // Return all NFTs owned by the given account
    @@throws(service-error) Page<Nft> findByOwner(ownerId: Address)

    // Return all NFTs of the given type
    @@throws(service-error) Page<Nft> findByType(tokenId: Address)

    // Return a single NFT identified by type and serial number
    @@throws(service-error) @@nullable Nft findByTypeAndSerial(tokenId: Address, serialNumber: int64)

    // Return all NFTs of the given type that are owned by the given account
    @@throws(service-error) Page<Nft> findByOwnerAndType(ownerId: Address, tokenId: Address)

    // Return a single NFT identified by owner, type, and serial number
    @@throws(service-error) @@nullable Nft findByOwnerAndTypeAndSerial(ownerId: Address, tokenId: Address, serialNumber: int64)
}

// Factory method to create the service (not needed for real framework integration where injection is used)
@@static
NftService createService(session: Session)
```

## Questions & Comments
