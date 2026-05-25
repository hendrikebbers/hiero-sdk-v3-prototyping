# Mirror Node Transaction Query API

## Description

## API Schema

```
namespace mirrornode.transaction
requires ledger, keys, mirrornode.common.

// All known transaction types. Each carries a protocolName matching the REST API wire value.
//TODO: That must be changed in future to make new services pluggable.
enum TransactionType {
    ACCOUNT_CREATE
    ACCOUNT_DELETE
    ACCOUNT_UPDATE
    CRYPTO_TRANSFER
    TOPIC_CREATE
    TOPIC_MESSAGE_SUBMIT
    TOKEN_CREATE
    TOKEN_MINT
    TOKEN_BURN
    TOKEN_TRANSFER
    CONTRACT_CREATE
    CONTRACT_CALL
    ETHEREUM
    ...              // full list derived from the Mirror Node OpenAPI spec
    UNKNOWN

    @@immutable protocolName: string
}

enum TransactionResult {
    SUCCESS
    FAIL
}

enum BalanceModification {
    CREDIT
    DEBIT
}

@@finalType
Transfer {
    @@immutable account: ledger.Address
    @@immutable amount: int64
    @@immutable isApproval: bool
}

@@finalType
TransactionInfo {
    @@immutable transactionId: string
    @@immutable transactionHash: bytes
    @@immutable chargedTxFee: int64
    @@immutable consensusTimestamp: zonedDateTime
    @@immutable @@nullable entityId: string
    @@immutable maxFee: int64
    @@immutable memo: bytes
    @@immutable name: TransactionType
    @@immutable nonce: int32
    @@immutable @@nullable node: string
    @@immutable @@nullable parentConsensusTimestamp: zonedDateTime
    @@immutable result: TransactionResult
    @@immutable scheduled: bool
    @@immutable validDurationSeconds: int64
    @@immutable validStartTimestamp: zonedDateTime
    @@immutable transfers: list<mirrornode.common.Transfer>
    @@immutable tokenTransfers: list<mirrornode.common.TokenTransfer>
    @@immutable nftTransfers: list<mirrornode.common.NftTransfer>
    @@immutable stakingRewardTransfers: list<mirrornode.common.StakingRewardTransfer>
}

abstraction TransactionRepository {
    @@async @@throws(mirror-node-error)
    Page<TransactionInfo> findByAccount(accountId: ledger.Address)

    @@async @@throws(mirror-node-error)
    Page<TransactionInfo> findByAccountAndType(accountId: ledger.Address, type: TransactionType)

    @@async @@throws(mirror-node-error)
    Page<TransactionInfo> findByAccountAndResult(accountId: ledger.Address, result: TransactionResult)

    @@async @@throws(mirror-node-error)
    Page<TransactionInfo> findByAccountAndModification(accountId: ledger.Address, modification: BalanceModification)

    @@async @@throws(mirror-node-error)
    @@nullable TransactionInfo findById(transactionId: string)
}

@static createRepository(mirrorNode: ledger.MirrorNode)

```
