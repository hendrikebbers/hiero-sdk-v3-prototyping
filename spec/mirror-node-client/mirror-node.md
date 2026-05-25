# Mirror Node Query API

## Description

This API defines the types and repository abstractions for querying a Hiero Mirror Node via its REST API.
It covers accounts, tokens, NFTs, transactions, topics, contracts, and network-level data.

## API Schema

```
namespace mirrornode

requires ledger, keys, mirrornode.account, mirrornode.contract, mirrornode.network, mirrornode.nft, mirrornode.token, mirrornode.topic, mirrornode.transaction

MirrorNodeClient {
    @@immutable accounts: AccountRepository
    @@immutable contracts: ContractRepository
    @@immutable network: NetworkRepository
    @@immutable network: NftRepository
    @@immutable network: TokenRepository
    @@immutable network: TopicRepository
    @@immutable network: TransactionRepository    
}

@@static
@@throws(mirror-node-error)
MirrorNodeClient createMirrorNodeClient(mirrorNode: ledger.MirrorNode)
```

## Example

```
mirrorNode = MirrorNode(restBaseUrl: "https://mainnet.mirrornode.hedera.com/api/v1")
client = await createMirrorNodeClient(mirrorNode)

// Look up an contract
contract = await client.contracts.findById(fromString("0.0.1234"))
```