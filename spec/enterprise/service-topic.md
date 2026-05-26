# Topic Service API

Service definition for the Consensus Service: creating topics and submitting messages.

## Description

Provides high-level write operations for consensus topics: creating public or private (submit-key protected) topics,
updating their keys and memo, deleting them, and submitting messages. Topic ids are `ledger.Address` values.

Reading topic messages (history and live subscription) is **not** part of this write-oriented service — it is
covered by the mirror-node query API (`mirrornode.topic`, see `spec/mirror-node-client/mirror-node-topic.md`).

## API Schema

```
namespace enterprise.service.topic
requires {Address} from ledger
requires {NetworkSetting} from ledger.config
requires {PublicKey} from keys
requires {Account, TransactionSigner} from consensusnode.client

TopicService {

    // Create a public topic (anyone may submit messages). An admin key allows later updates/deletion.
    @@throws(service-error) Address createTopic()

    @@throws(service-error) Address createTopic(memo: string)

    @@throws(service-error) Address createTopic(adminKey: PublicKey, memo: string)

    // Delete a topic (requires admin-key authority)
    @@throws(service-error) void deleteTopic(topicId: Address)

    // Submit a message to a topic
    @@throws(service-error) void submitMessage(topicId: Address, message: bytes)
}

// Factory methods to create the service (not needed for real framework integration where injection is used)
@@static
TopicService createService(networkSettings: NetworkSetting, operatorAccount: Account)

@@static
TopicService createService(networkSettings: NetworkSetting, operatorAccount: Account, transactionSigner: TransactionSigner)
```

## Questions & Comments
