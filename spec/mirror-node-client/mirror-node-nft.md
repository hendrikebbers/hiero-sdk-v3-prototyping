# Mirror Node NFT Query API

## Description

## API Schema

```
namespace mirrornode.nft
requires {Address, MirrorNode} from ledger

@@finalType
NftMetadata {
    @@immutable tokenId: Address
    @@immutable name: string
    @@immutable symbol: string
    @@immutable treasuryAccountId: Address
}

@@finalType
Nft {
    @@immutable tokenId: Address
    @@immutable serial: int64
    @@immutable owner: Address
    @@immutable metadata: bytes
}

abstraction NftRepository {
    @@async @@throws(mirror-node-error)
    Page<Nft> findByOwner(ownerId: Address)

    @@async @@throws(mirror-node-error)
    Page<Nft> findByType(tokenId: Address)

    @@async @@throws(mirror-node-error)
    @@nullable Nft findByTypeAndSerial(tokenId: Address, serialNumber: int64)

    @@async @@throws(mirror-node-error)
    Page<Nft> findByOwnerAndType(ownerId: Address, tokenId: Address)

    @@async @@throws(mirror-node-error)
    Page<NftMetadata> findAllTypes()

    @@async @@throws(mirror-node-error)
    Page<NftMetadata> findTypesByOwner(ownerId: Address)

    @@async @@throws(mirror-node-error)
    @@nullable NftMetadata getNftMetadata(tokenId: Address)
}

@static createRepository(mirrorNode: MirrorNode)

```
