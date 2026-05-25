# Mirror Node NFT Query API

## Description

## API Schema

```
namespace mirrornode.nft
requires ledger, keys, mirrornode.common

@@finalType
NftMetadata {
    @@immutable tokenId: ledger.Address
    @@immutable name: string
    @@immutable symbol: string
    @@immutable treasuryAccountId: ledger.Address
}

@@finalType
Nft {
    @@immutable tokenId: ledger.Address
    @@immutable serial: int64
    @@immutable owner: ledger.Address
    @@immutable metadata: bytes
}

abstraction NftRepository {
    @@async @@throws(mirror-node-error)
    Page<Nft> findByOwner(ownerId: ledger.Address)

    @@async @@throws(mirror-node-error)
    Page<Nft> findByType(tokenId: ledger.Address)

    @@async @@throws(mirror-node-error)
    @@nullable Nft findByTypeAndSerial(tokenId: ledger.Address, serialNumber: int64)

    @@async @@throws(mirror-node-error)
    Page<Nft> findByOwnerAndType(ownerId: ledger.Address, tokenId: ledger.Address)

    @@async @@throws(mirror-node-error)
    Page<NftMetadata> findAllTypes()

    @@async @@throws(mirror-node-error)
    Page<NftMetadata> findTypesByOwner(ownerId: ledger.Address)

    @@async @@throws(mirror-node-error)
    @@nullable NftMetadata getNftMetadata(tokenId: ledger.Address)
}

@static createRepository(mirrorNode: ledger.MirrorNode)

```
