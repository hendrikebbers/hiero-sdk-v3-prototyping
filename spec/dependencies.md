# Module / Namespace Dependencies

This document gives an overview of how the V3 API modules (namespaces) depend on each other. The graphs are derived
from the `requires {Type} from <namespace>` declarations in the spec files — an edge `A --> B` means "namespace `A`
imports one or more types from namespace `B`".

The namespaces group into the five `spec/` folders / layers:

- **base** — `common`, `grpc`, `proto`, `nativeToken`, `keys`, `ledger`, `ledger.config`, `endorsement`, `hedera`
- **consensus-node-client** — `consensusnode.client`, `consensusnode.transactions[.accounts|.spi]`, `consensusnode.queries[.accounts|.files]`, `consensusnode.proto[.account]`
- **consensus-node-admin-client** — `consensusnode.admin.{freeze|system|nodes|network}`
- **mirror-node-client** — `mirrornode` and its per-domain sub-namespaces
- **enterprise** — `enterprise.service[.common|.account|.contract|.file|.token|.nft|.topic]`

## Layer overview

```mermaid
flowchart LR
    enterprise["enterprise<br/>(high-level service layer)"]
    consensus["consensus-node-client"]
    admin["consensus-node-admin-client<br/>(privileged council ops)"]
    mirror["mirror-node-client"]
    base["base<br/>(common, ledger, keys, nativeToken, …)"]

    enterprise --> consensus
    enterprise --> mirror
    enterprise --> base
    consensus --> base
    admin --> consensus
    admin --> base
    mirror --> base
```

`mirror-node-client` and `consensus-node-client` are independent of each other;
`consensus-node-admin-client` is opt-in on top of `consensus-node-client` (privileged
freeze / system-delete / DAB-node transactions that normal applications do not consume);
`enterprise` builds on top of `consensus-node-client`, `mirror-node-client`, and `base`.
Everything ultimately rests on `base`.

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
        endorsement["endorsement"]
        hedera["hedera"]
    end

    subgraph consensus["consensus-node-client"]
        cn_client["consensusnode.client"]
        cn_tx["consensusnode.transactions"]
        cn_tx_accounts["…transactions.accounts"]
        cn_tx_spi["…transactions.spi"]
        cn_queries["consensusnode.queries"]
        cn_queries_accounts["…queries.accounts"]
        cn_queries_files["…queries.files"]
        cn_proto["consensusnode.proto"]
        cn_proto_account["…proto.account ∅"]
    end

    subgraph admin["consensus-node-admin-client"]
        cn_admin_freeze["consensusnode.admin.freeze"]
        cn_admin_system["consensusnode.admin.system"]
        cn_admin_nodes["consensusnode.admin.nodes"]
        cn_admin_network["consensusnode.admin.network"]
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
        ent_service["enterprise.service"]
        ent_common["…service.common"]
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
    endorsement --> keys
    endorsement --> ledger

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

    %% consensus-node-client (queries)
    cn_queries --> ledger
    cn_queries --> cn_client
    cn_queries --> nativeToken
    cn_queries_accounts --> ledger
    cn_queries_accounts --> keys
    cn_queries_accounts --> nativeToken
    cn_queries_accounts --> cn_queries
    cn_queries_files --> ledger
    cn_queries_files --> keys
    cn_queries_files --> cn_queries

    %% consensus-node-admin-client
    cn_admin_freeze --> ledger
    cn_admin_freeze --> cn_tx
    cn_admin_system --> ledger
    cn_admin_system --> cn_tx
    cn_admin_nodes --> ledger
    cn_admin_nodes --> keys
    cn_admin_nodes --> cn_tx
    cn_admin_network --> ledger
    cn_admin_network --> cn_queries
    cn_admin_network --> cn_admin_nodes

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
    ent_service --> ledger_config
    ent_service --> cn_client
    ent_account --> common
    ent_account --> ledger
    ent_account --> keys
    ent_account --> nativeToken
    ent_account --> ent_service
    ent_contract --> common
    ent_contract --> ledger
    ent_contract --> ent_service
    ent_file --> ledger
    ent_file --> ent_service
    ent_token --> common
    ent_token --> ledger
    ent_token --> keys
    ent_token --> mn_token
    ent_token --> ent_service
    ent_nft --> common
    ent_nft --> ledger
    ent_nft --> keys
    ent_nft --> mn_nft
    ent_nft --> ent_service
    ent_topic --> common
    ent_topic --> ledger
    ent_topic --> keys
    ent_topic --> ent_service

    classDef base fill:#e8f0fe,stroke:#4285f4,color:#000;
    classDef consensus fill:#e6f4ea,stroke:#34a853,color:#000;
    classDef admin fill:#d2e3fc,stroke:#1967d2,color:#000;
    classDef mirror fill:#fef7e0,stroke:#f9ab00,color:#000;
    classDef enterprise fill:#fce8e6,stroke:#ea4335,color:#000;

    class common,grpc,proto,nativeToken,keys,ledger,ledger_config,endorsement,hedera base;
    class cn_client,cn_tx,cn_tx_accounts,cn_tx_spi,cn_queries,cn_queries_accounts,cn_queries_files,cn_proto,cn_proto_account consensus;
    class cn_admin_freeze,cn_admin_system,cn_admin_nodes,cn_admin_network admin;
    class mn,mn_account,mn_common,mn_contract,mn_network,mn_nft,mn_token,mn_topic,mn_transaction mirror;
    class ent_service,ent_common,ent_account,ent_contract,ent_file,ent_token,ent_nft,ent_topic enterprise;
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
  domain types (`Nft`, `NftMetadata`, `Token`, `TokenInfo`, `Balance`) for its query methods instead of redeclaring
  them. (`enterprise.service.contract` keeps a thin local `Contract` type and therefore does not depend on
  `mirrornode.contract`.)
- **Admin layer is opt-in and one-way:** `consensus-node-admin-client` depends on `consensus-node-client` (it
  reuses `Transaction<$$Receipt>`, `Receipt`, and the signing / packing lifecycle) but is not depended on by it —
  the edge is strictly `admin --> consensus-node-client`. Applications that never call freeze / system-delete / DAB
  node operations do not pull the admin module in. `enterprise` does not depend on the admin module either; surfacing
  these privileged operations through the high-level service layer is intentionally out of scope.
- **Session as the shared root of `enterprise`:** `enterprise.service` defines the `Session` (network settings,
  operator account, transaction signer, read-your-writes high-water-mark) that every concrete service
  (`…service.account`, `…service.contract`, `…service.file`, `…service.token`, `…service.nft`, `…service.topic`)
  depends on. As a result the concrete services no longer import `ledger.config` or `consensusnode.client`
  directly — both reach them through `enterprise.service`.
- **Empty stubs (no edges):** `proto` and `consensusnode.proto.account` define no types yet. `consensusnode.proto`
  is used only by `consensusnode.transactions.spi`.
- **Isolated nodes (no edges):** `enterprise.service.common` defines a `Subscription` type but is not yet imported
  by any other namespace; the streaming methods on `…service.topic` would be the natural caller once they return
  the subscription handle.
- **Hedera is base-resident:** `hedera` (HBAR + Hedera network settings) currently lives in the base layer and depends
  on `ledger.config` and `nativeToken` — see the open question in
  [`native-token.md`](base/native-token.md) about whether Hedera-specific namespaces should move out of `base`.

> The graphs are generated from the `requires` declarations; regenerate after changing imports.
