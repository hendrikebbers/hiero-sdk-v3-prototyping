# Mirror Node NFT Query API

## Description

## API Schema

```
namespace mirrornode.nft
requires {Address, AccountId, MirrorNode} from ledger
requires {Page} from common

@@finalType
NftMetadata {
    @@immutable tokenId: Address
    @@immutable name: string
    @@immutable symbol: string
    @@immutable treasuryAccountId: AccountId
}

@@finalType
Nft {
    @@immutable tokenId: Address
    @@immutable serial: int64
    @@immutable owner: AccountId
    @@immutable metadata: bytes
}

abstraction NftRepository {
    @@async @@throws(mirror-node-error)
    Page<Nft> findByOwner(ownerId: AccountId)

    @@async @@throws(mirror-node-error)
    Page<Nft> findByType(tokenId: Address)

    @@async @@throws(mirror-node-error)
    @@nullable Nft findByTypeAndSerial(tokenId: Address, serialNumber: int64)

    @@async @@throws(mirror-node-error)
    Page<Nft> findByOwnerAndType(ownerId: AccountId, tokenId: Address)

    @@async @@throws(mirror-node-error)
    Page<NftMetadata> findAllTypes()

    @@async @@throws(mirror-node-error)
    Page<NftMetadata> findTypesByOwner(ownerId: AccountId)

    @@async @@throws(mirror-node-error)
    @@nullable NftMetadata getNftMetadata(tokenId: Address)
}

@@static NftRepository createRepository(mirrorNode: MirrorNode)

```
