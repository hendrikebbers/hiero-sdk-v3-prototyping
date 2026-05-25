# Configuration API

This section defines the API for configuration.

## Description

The config API provides functions to define and retrieve the configuration of a specific network.

## API Schema

```
namespace ledger.config
requires ledger

constant HEDERA_MAINNET_IDENTIFIER:string = "hedera-mainnet" // identifier for the Hedera mainnet
constant HEDERA_TESTNET_IDENTIFIER:string = "hedera-testnet" // identifier for the Hedera testnet

// The full configuration to connect to a specific network
NetworkSetting {
 
    @@immutable ledger: ledger.Ledger // the definition of the ledger
   
    // Returns an immutable set of consensus nodes
    // Modifications to the returned set do not affect the original
    @@immutable set<ledger.ConsensusNode> getConsensusNodes()

    // Returns an immutable set of mirror nodes
    // Modifications to the returned set do not affect the original
    @@immutable set<ledger.MirrorNode> getMirrorNodes()

}

// factory methods of `NetworkSetting` that should be added to the namespace in the best language dependent way

// Method to register a network configuration
@@static void registerNetworkSetting(identifier: string, setting: NetworkSetting)

// throws not-found-error if no network with that identifier exists
// Network settings can be added as plug and play by external modules
@@throws(not-found-error) @@static NetworkSetting getNetworkSetting(identifier: string) 
```

## Examples

The following example shows how to load the network configuration for the Hedera testnet:

```
NetworkSetting setting = NetworkSetting.getNetworkSetting(HEDERA_TESTNET_IDENTIFIER)
```

## Questions & Comments
