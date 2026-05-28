# Module / Namespace Dependencies

This document gives an overview of how the V3 API modules (namespaces) depend on each other. The graphs are derived
from the `requires {Type} from <namespace>` declarations in the spec files — an edge `A --> B` means "namespace `A`
imports one or more types from namespace `B`".

The namespaces group into the four `spec/` folders / layers:

- **base** — `common`, `grpc`, `proto`, `nativeToken`, `keys`, `ledger`, `ledger.config`, `hedera`
- **consensus-node-client** — `consensusnode.client`, `consensusnode.transactions[.accounts|.spi]`, `consensusnode.proto[.account]`
- **mirror-node-client** — `mirrornode` and its per-domain sub-namespaces
- **enterprise** — `enterprise.service[.account|.contract|.file|.token|.nft|.topic]`

## Layer overview

```mermaid
flowchart LR
    enterprise["enterprise<br/>(high-level service layer)"]
    consensus["consensus-node-client"]
    mirror["mirror-node-client"]
    base["base<br/>(common, ledger, keys, nativeToken, …)"]

    enterprise --> consensus
    enterprise --> mirror
    enterprise --> base
    consensus --> base
    mirror --> base
```

`mirror-node-client` and `consensus-node-client` are independent of each other; `enterprise` builds on top of
`consensus-node-client`, `mirror-node-client`, and `base`. Everything ultimately rests on `base`.

## Namespace dependency graph

```mermaid
flowchart LR
    subgraph base["base"]
        common["common"]
        grpc["grpc"]
        proto["proto ∅"]
        nativeToken["nativeToken"]
        keys["keys"]
        ledger["ledger"]
        ledger_config["ledger.config"]
        hedera["hedera"]
    end

    subgraph consensus["consensus-node-client"]
        cn_client["consensusnode.client"]
        cn_tx["consensusnode.transactions"]
        cn_tx_accounts["…transactions.accounts"]
        cn_tx_spi["…transactions.spi"]
        cn_proto["consensusnode.proto"]
        cn_proto_account["…proto.account ∅"]
    end

    subgraph mirror["mirror-node-client"]
        mn["mirrornode"]
        mn_account["mirrornode.account"]
        mn_common["mirrornode.common"]
        mn_contract["mirrornode.contract"]
        mn_network["mirrornode.network"]
        mn_nft["mirrornode.nft"]
        mn_token["mirrornode.token"]
        mn_topic["mirrornode.topic"]
        mn_transaction["mirrornode.transaction"]
    end

    subgraph enterprise["enterprise"]
        ent_service["enterprise.service ∅"]
        ent_account["…service.account"]
        ent_contract["…service.contract"]
        ent_file["…service.file"]
        ent_token["…service.token"]
        ent_nft["…service.nft"]
        ent_topic["…service.topic"]
    end

    %% base-internal
    hedera --> ledger_config
    hedera --> nativeToken
    ledger_config --> ledger
    ledger --> nativeToken

    %% consensus-node-client
    cn_client --> ledger
    cn_client --> ledger_config
    cn_client --> keys
    cn_client --> nativeToken
    cn_tx --> ledger
    cn_tx --> nativeToken
    cn_tx --> cn_client
    cn_tx_accounts --> ledger
    cn_tx_accounts --> keys
    cn_tx_accounts --> nativeToken
    cn_tx_accounts --> cn_tx
    cn_tx_spi --> cn_tx
    cn_tx_spi --> grpc
    cn_tx_spi --> cn_proto

    %% mirror-node-client
    mn --> ledger
    mn --> mn_account
    mn --> mn_contract
    mn --> mn_network
    mn --> mn_nft
    mn --> mn_token
    mn --> mn_topic
    mn --> mn_transaction
    mn_account --> ledger
    mn_account --> keys
    mn_account --> common
    mn_common --> ledger
    mn_contract --> ledger
    mn_contract --> keys
    mn_contract --> common
    mn_network --> ledger
    mn_network --> common
    mn_nft --> ledger
    mn_nft --> common
    mn_token --> ledger
    mn_token --> mn_common
    mn_token --> common
    mn_topic --> ledger
    mn_topic --> keys
    mn_topic --> mn_common
    mn_topic --> common
    mn_transaction --> ledger
    mn_transaction --> mn_common
    mn_transaction --> common

    %% enterprise
    ent_account --> common
    ent_account --> ledger
    ent_account --> ledger_config
    ent_account --> keys
    ent_account --> nativeToken
    ent_account --> cn_client
    ent_contract --> common
    ent_contract --> ledger
    ent_contract --> ledger_config
    ent_contract --> cn_client
    ent_contract --> mn_contract
    ent_file --> ledger
    ent_file --> ledger_config
    ent_file --> cn_client
    ent_token --> common
    ent_token --> ledger
    ent_token --> ledger_config
    ent_token --> keys
    ent_token --> cn_client
    ent_token --> mn_token
    ent_nft --> common
    ent_nft --> ledger
    ent_nft --> ledger_config
    ent_nft --> keys
    ent_nft --> cn_client
    ent_nft --> mn_nft
    ent_topic --> common
    ent_topic --> ledger
    ent_topic --> ledger_config
    ent_topic --> keys
    ent_topic --> cn_client

    classDef base fill:#e8f0fe,stroke:#4285f4,color:#000;
    classDef consensus fill:#e6f4ea,stroke:#34a853,color:#000;
    classDef mirror fill:#fef7e0,stroke:#f9ab00,color:#000;
    classDef enterprise fill:#fce8e6,stroke:#ea4335,color:#000;

    class common,grpc,proto,nativeToken,keys,ledger,ledger_config,hedera base;
    class cn_client,cn_tx,cn_tx_accounts,cn_tx_spi,cn_proto,cn_proto_account consensus;
    class mn,mn_account,mn_common,mn_contract,mn_network,mn_nft,mn_token,mn_topic,mn_transaction mirror;
    class ent_service,ent_account,ent_contract,ent_file,ent_token,ent_nft,ent_topic enterprise;
```

`∅` marks namespaces that are still empty stubs (no types defined yet).

## Observations

- **Foundation:** `nativeToken` and `ledger` are the most depended-upon base namespaces; `common` (the `Page<$$T>`
  type) is required by most mirror-node namespaces and by every enterprise service that exposes paginated queries
  (`account`, `topic`, `nft`, `token`, `contract`).
- **No cycles:** the dependency graph is acyclic. (The earlier `keys ↔ keys.io` cycle was removed by merging the
  `keys.io` sub-namespace into `keys`.)
- **Clean layer separation:** `mirror-node-client` has no dependency on `consensus-node-client` (or vice versa), and
  `base` depends on nothing outside `base`. `enterprise` is the only layer that combines `consensus-node-client`,
  `mirror-node-client`, and `base` — the mirror-node edge was introduced so that the service layer can reuse the
  domain types (`Nft`, `NftMetadata`, `Token`, `TokenInfo`, `Balance`, `Contract`) for its query methods instead of
  redeclaring them.
- **Empty stubs (no edges):** `proto`, `consensusnode.proto.account`, and `enterprise.service` define no types yet.
  `consensusnode.proto` is used only by `consensusnode.transactions.spi`.
- **Hedera is base-resident:** `hedera` (HBAR + Hedera network settings) currently lives in the base layer and depends
  on `ledger.config` and `nativeToken` — see the open question in
  [`native-token.md`](base/native-token.md) about whether Hedera-specific namespaces should move out of `base`.

> The graphs are generated from the `requires` declarations; regenerate after changing imports.
