# Client API

This section defines the client API.

## Description

The client API is the API that will be used by the SDK to interact with the network.
A client defines a concrete network connection to a specific network with a specific operator account.

## API Schema

```
namespace consensusnode.client
requires {Address, Ledger} from ledger
requires {NetworkSetting} from ledger.config
requires {PrivateKey} from keys
requires {NativeTokenUnit} from nativeToken

// Definition of an account that signs and pays for requests
Account {
    @@immutable accountId: Address // the account id of the operator
    @@immutable privateKey: PrivateKey // the private key of the operator
}

// Helper to allow external signing of transactions
abstraction TransactionSigner {
  bytes signTransaction(transactionBytes: bytes) // returns the signature as a byte array
}

// The client API that will be used by the SDK to interact with the network
HieroClient<$$Unit extends NativeTokenUnit> {
    @@immutable operatorAccount: Account // the operator account
    @@immutable ledger: Ledger<$$Unit> // the network to connect to
    @@immutable transactionSigner: TransactionSigner // by default the operator account is used, but this allows to use an external signer for transactions
    // TO_BE_DEFINED_IN_FUTURE_VERSIONS
}

// factory methods of `HieroClient` that should be added to the namespace in the best language dependent way

@@static HieroClient<ANY> createClient(networkSettings: NetworkSetting, operatorAccount: Account)
@@static HieroClient<ANY> createClient(networkSettings: NetworkSetting, operatorAccount: Account, transactionSigner: TransactionSigner)
```

## Examples

The following example shows how to create a `HieroClient` instance:

```
Address accountId = ...;
PrivateKey privateKey = ...;
Account operatorAccount = new Account(accountId, privateKey);

NetworkSetting networkSettings = ...;

HieroClient client = HieroClient.createClient(networkSettings, operatorAccount);
```

## Questions & Comments

