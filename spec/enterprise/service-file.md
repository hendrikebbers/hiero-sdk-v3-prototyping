# File Service API

Service definition for storing and retrieving files on a Hiero network (the File Service).

## Description

Files hold an arbitrary byte payload on the ledger together with an expiration time. The service abstracts the
chunking that is required for payloads larger than a single transaction. Entity ids are `ledger.Address` values.

## API Schema

```
namespace enterprise.service.file
requires {Address} from ledger
requires {NetworkSetting} from ledger.config
requires {Account, TransactionSigner} from consensusnode.client

FileService {

    // Create a new file with the given contents; returns the id of the new file
    @@throws(service-error) Address createFile(contents: bytes)

    @@throws(service-error) Address createFile(contents: bytes, expirationTime: zonedDateTime)

    // Read the full contents of a file
    @@throws(service-error) bytes readFile(fileId: Address)

    // Replace the contents of a file
    @@throws(service-error) void updateFile(fileId: Address, contents: bytes)

    // Update only the expiration time of a file
    @@throws(service-error) void updateExpirationTime(fileId: Address, expirationTime: zonedDateTime)

    // Delete a file
    @@throws(service-error) void deleteFile(fileId: Address)

    // Returns true if the file has been deleted
    @@throws(service-error) bool isDeleted(fileId: Address)

    // Size of the file contents in bytes
    @@throws(service-error) int32 getSize(fileId: Address)

    // Expiration time of the file
    @@throws(service-error) zonedDateTime getExpirationTime(fileId: Address)
}

// Factory methods to create the service (not needed for real framework integration where injection is used)
@@static
FileService createService(networkSettings: NetworkSetting, operatorAccount: Account)

@@static
FileService createService(networkSettings: NetworkSetting, operatorAccount: Account, transactionSigner: TransactionSigner)
```

## Questions & Comments
