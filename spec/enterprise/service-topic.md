# Topic Service API


## Description

## API Schema

```
namespace enterprise.service.topic
requires {Page} from common
requires {Address} from ledger
requires {PublicKey} from keys
requires {Session} from enterprise.service

@@finalType
Topic {
    @@immutable topicId: Address
    @@immutable @@nullable adminKey: PublicKey
    @@immutable @@nullable submitKey: PublicKey
    @@immutable createdTimestamp: zonedDateTime
    @@immutable memo: string
}

@@finalType
TopicMessage {
    @@immutable consensusTimestamp: zonedDateTime
    @@immutable message: string
    @@immutable payerAccountId: Address
    @@immutable sequenceNumber: int64
    @@immutable topicId: Address
}

TopicService {

    // Create a public topic (anyone may submit messages). An admin key allows later updates/deletion.
    @@throws(service-error) Topic createTopic()

    @@throws(service-error) Topic createTopic(memo: string)

    @@throws(service-error) Topic createTopic(adminKey: PublicKey, memo: string)
    
    @@throws(service-error) @@nullable Topic findById(topicId: Address)
    
    @@throws(service-error) Page<Topic> getAll()

    // Delete a topic (requires admin-key authority)
    @@throws(service-error) void deleteTopic(topicId: Address)

    // Submit a message to a topic
    @@throws(service-error) TopicMessage submitMessage(topicId: Address, message: bytes)
    
    @@throws(service-error) Page<TopicMessage> getMessages(topicId: Address)
    
    @@throws(service-error) TopicMessage getMessageBySequenceNumber(topicId: Address, sequenceNumber: int64)
    
    @@streaming TopicMessage subscribe(topicId: Address)
}

// Factory method to create the service (not needed for real framework integration where injection is used)
@@static
TopicService createService(session: Session)
```

## Questions & Comments
