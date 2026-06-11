# Mirror Node Topic Query API

## Description

## API Schema

```
namespace mirrornode.topic
requires {Address, MirrorNode, TransactionId} from ledger
requires {PublicKey} from keys
requires {FixedFee} from mirrornode.common
requires {Page} from common

@@finalType
Topic {
    @@immutable topicId: Address
    @@immutable @@nullable adminKey: PublicKey
    @@immutable @@nullable submitKey: PublicKey
    @@immutable @@nullable feeScheduleKey: PublicKey
    @@immutable @@nullable autoRenewAccount: Address
    @@immutable autoRenewPeriod: seconds
    @@immutable createdTimestamp: zonedDateTime
    @@immutable deleted: bool
    @@immutable memo: string
    @@immutable fixedFees: list<FixedFee>
    @@immutable @@nullable feeExemptKeyList: list<PublicKey>
    @@immutable fromTimestamp: zonedDateTime
    @@immutable toTimestamp: zonedDateTime
}

@@finalType
ChunkInfo {
    @@immutable initialTransactionId: TransactionId
    @@immutable nonce: int32
    @@immutable number: int32
    @@immutable total: int32
    @@immutable scheduled: bool
}

@@finalType
TopicMessage {
    @@immutable @@nullable chunkInfo: ChunkInfo
    @@immutable consensusTimestamp: zonedDateTime
    @@immutable message: string
    @@immutable payerAccountId: Address
    @@immutable runningHash: bytes
    @@immutable runningHashVersion: int32
    @@immutable sequenceNumber: int64
    @@immutable topicId: Address
}

abstraction TopicRepository {
    @@async @@throws(mirror-node-error)
    @@nullable Topic findById(topicId: Address)

    @@async @@throws(mirror-node-error)
    Page<TopicMessage> getMessages(topicId: Address)

    @@async @@throws(mirror-node-error)
    @@nullable TopicMessage getMessageBySequenceNumber(topicId: Address, sequenceNumber: int64)
}

@@static TopicRepository createRepository(mirrorNode: MirrorNode)

```
