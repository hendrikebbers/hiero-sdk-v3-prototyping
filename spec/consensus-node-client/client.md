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

type NodeSignature {
      @@immutable node: Address
      @@immutable publicKey: PublicKey
      @@immutable signature: bytes
}

// Helper to allow external signing of transactions
abstraction TransactionSigner {

  NodeSignature signTransaction(transactionBytes: bytes, node: Address)
}

// Common base for anything that is handed off to the consensus node network and
// produces a typed result — both queries (consensusnode.queries.Query) and
// packed transactions (consensusnode.transactions.PackedTransaction) extend this.
// Carries the shared retry-tuning knobs (max attempts, backoff window, per-attempt
// timeout) and the single submit() entry point. The SDK applies these settings
// when selecting a consensus node, retrying transient gRPC failures, and bounding
// the total wait.
abstraction Submittable<$$Result> {

    @@nullable maxAttempts: int32
    @@nullable maxBackoff: int64
    @@nullable minBackoff: int64
    @@nullable attemptTimeout: int64

    // Hand off to the network and return the typed result. Node selection,
    // retry, and any operation-specific protocol details (e.g. cost discovery
    // and payment for PaidQuery) are handled by the SDK transparently.
    @@async $$Result submit(client: HieroClient)
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

