# Mirror Node Transaction Query API

## Description

## API Schema

```
namespace mirrornode.transaction
requires {AccountId, MirrorNode, TransactionId} from ledger
requires {NftTransfer, StakingRewardTransfer, TokenTransfer, Transfer} from mirrornode.common
requires {Page} from common

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
TransactionInfo {
    @@immutable transactionId: TransactionId
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
    @@immutable validDuration: seconds
    @@immutable validStartTimestamp: zonedDateTime
    @@immutable transfers: list<Transfer>
    @@immutable tokenTransfers: list<TokenTransfer>
    @@immutable nftTransfers: list<NftTransfer>
    @@immutable stakingRewardTransfers: list<StakingRewardTransfer>
}

abstraction TransactionRepository {
    @@async @@throws(mirror-node-error)
    Page<TransactionInfo> findByAccount(accountId: AccountId)

    @@async @@throws(mirror-node-error)
    Page<TransactionInfo> findByAccountAndType(accountId: AccountId, type: TransactionType)

    @@async @@throws(mirror-node-error)
    Page<TransactionInfo> findByAccountAndResult(accountId: AccountId, result: TransactionResult)

    @@async @@throws(mirror-node-error)
    Page<TransactionInfo> findByAccountAndModification(accountId: AccountId, modification: BalanceModification)

    @@async @@throws(mirror-node-error)
    @@nullable TransactionInfo findById(transactionId: TransactionId)
}

@@static TransactionRepository createRepository(mirrorNode: MirrorNode)

```
