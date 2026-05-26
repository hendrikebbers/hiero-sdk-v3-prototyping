# Configuration API

This section defines the API for configuration.

## Description

The config API provides functions to define and retrieve the configuration of a specific network.

## API Schema

```
namespace ledger.config
requires {Ledger, ConsensusNode, MirrorNode} from ledger

// The full configuration to connect to a specific network
NetworkSetting {
 
    @@immutable ledger: Ledger
   
    @@immutable  getConsensusNodes: set<ConsensusNode>

    @@immutable  getMirrorNodes: set<MirrorNode>

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
