# Mirror Node Topic Query API

## Description

## API Schema

```
namespace mirrornode.topic
requires ledger, keys, mirrornode.common

@@finalType
Topic {
    @@immutable topicId: ledger.Address
    @@immutable @@nullable adminKey: keys.PublicKey
    @@immutable @@nullable submitKey: keys.PublicKey
    @@immutable @@nullable feeScheduleKey: keys.PublicKey
    @@immutable @@nullable autoRenewAccount: ledger.Address
    @@immutable autoRenewPeriod: int32
    @@immutable createdTimestamp: zonedDateTime
    @@immutable deleted: bool
    @@immutable memo: string
    @@immutable fixedFees: list<mirrornode.common.FixedFee>
    @@immutable @@nullable feeExemptKeyList: list<keys.PublicKey>
    @@immutable fromTimestamp: zonedDateTime
    @@immutable toTimestamp: zonedDateTime
}

@@finalType
ChunkInfo {
    @@immutable initialTransactionId: ledger.TransactionId
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
    @@immutable payerAccountId: ledger.Address
    @@immutable runningHash: bytes
    @@immutable runningHashVersion: int32
    @@immutable sequenceNumber: int64
    @@immutable topicId: ledger.Address
}

abstraction TopicRepository {
    @@async @@throws(mirror-node-error)
    @@nullable Topic findById(topicId: ledger.Address)

    @@async @@throws(mirror-node-error)
    Page<TopicMessage> getMessages(topicId: ledger.Address)

    @@async @@throws(mirror-node-error)
    @@nullable TopicMessage getMessageBySequenceNumber(topicId: ledger.Address, sequenceNumber: int64)
}

@static createRepository(mirrorNode: ledger.MirrorNode)

```
