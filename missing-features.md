# Missing Features in the V3 Specification

This document tracks features and APIs that exist in the current Hedera SDK v2
(and the surrounding Hedera/Hiero documentation) but are not yet covered by
the V3 specification under [`spec/`](spec/). It is intended as a working
backlog — not a final scope statement. Items may move, get reprioritized, or
be intentionally excluded for V3.

Status legend:

- :x: not specified
- :warning: partially specified
- :white_check_mark: specified

---

## 1. `consensus-node-client` — transactions and queries

Only the **account service** is currently specified
(`AccountCreate/Update/Delete`). All other service transactions are missing.

### 1.1 Crypto / transfers

| API in v2 | V3 spec |
| --- | --- |
| `TransferTransaction` (HBAR + fungible + NFT in one) | :white_check_mark: |
| `AccountAllowanceApproveTransaction` (HBAR / token / NFT / all serials) | :white_check_mark: |
| `AccountAllowanceDeleteTransaction` | :white_check_mark: |

### 1.2 Token service (HTS) — **entire service missing**

`TokenCreate`, `TokenUpdate`, `TokenDelete`, `TokenAssociate`, `TokenDissociate`,
`TokenMint`, `TokenBurn`, `TokenWipe`, `TokenFreeze` / `TokenUnfreeze`,
`TokenGrantKyc` / `TokenRevokeKyc`, `TokenPause` / `TokenUnpause`,
`TokenFeeScheduleUpdate`, `TokenUpdateNfts` (HIP-657),
`TokenAirdrop` / `TokenClaimAirdrop` / `TokenCancelAirdrop` /
`TokenReject` (HIP-904) — **all :x:**

### 1.3 File service

`FileCreate`, `FileAppend` (chunked), `FileUpdate`, `FileDelete` — :white_check_mark:
([`transactions-files.md`](spec/consensus-node-client/transactions-files.md)).
Open: chunked-append model (per-chunk receipts vs. single-receipt facade) — tracked in that
file's *Questions & Comments*.

### 1.4 Smart contract service

`ContractCreate`, `ContractCreateFlow`, `ContractUpdate`, `ContractExecute`,
`ContractDelete`, `EthereumTransaction` — **all :x:**

### 1.5 Topic / HCS

`TopicCreate`, `TopicUpdate`, `TopicDelete`, `TopicMessageSubmit` (chunked) —
**all :x:**. HIP-991 custom fees on topics (`customFees`, `feeScheduleKey`,
`feeExemptKeyList`) — :x:

### 1.6 Schedule service (HIP-755 long-term)

`ScheduleCreate`, `ScheduleSign`, `ScheduleDelete` + `customFeeLimits` —
**all :x:**

### 1.7 System / network admin

`FreezeTransaction` (`FREEZE_ONLY`, `PREPARE_UPGRADE`, `FREEZE_UPGRADE`,
`FREEZE_ABORT`, `TELEMETRY_UPGRADE`), `SystemDelete`, `SystemUndelete` —
**all :x:**

### 1.8 Dynamic Address Book (HIP-869 / HIP-1064)

`NodeCreate`, `NodeUpdate`, `NodeDelete` with `gossipEndpoints`,
`serviceEndpoints`, `gossipCaCertificate`, `grpcCertificateHash`,
`grpcWebProxyEndpoint` (HIP-1046) — **all :x:**

### 1.9 Utility

- `PrngTransaction` (HIP-351) — :x:
- `BatchTransaction` (HIP-551, with `batchify()` on inner transactions) — :x:
- Hiero Hooks (HIP-1195): `HookStoreTransaction`, `EvmHook`, `EvmHookCall`,
  `HookCreationDetails`, ... — :x:

### 1.10 Queries (consensus-node-side)

Base abstractions are now specified in
[`spec/consensus-node-client/queries.md`](spec/consensus-node-client/queries.md):

- `Query<$$Result>` — free queries; extends `Submittable<QueryResponse<$$Result>>`.
- `PaidQuery<$$Result>` — paid queries; adds auto-discovered cost, `maxQueryPayment`
  ceiling, optional `payer` override (sponsored queries), and `getCost()`.
- `QueryResponse<$$T>` / `PaidQueryResponse<$$T>` — envelope types carrying `value`,
  `answeredBy`, and (paid only) `cost`.

First concrete service file landed in
[`queries-accounts.md`](spec/consensus-node-client/queries-accounts.md):

| Query | V3 spec |
| --- | --- |
| `AccountBalanceQuery` (free) | :white_check_mark: |
| `AccountInfoQuery` (paid) | :white_check_mark: |

Still missing per service:

- `AccountRecordsQuery`
- `ContractCallQuery` (local EVM call), `ContractInfoQuery`, `ContractBytecodeQuery`
- `FileContentsQuery`, `FileInfoQuery`
- `TokenInfoQuery`, `TokenNftInfoQuery`
- `TopicInfoQuery`
- `ScheduleInfoQuery`
- `NetworkVersionInfoQuery`, `NodeAddressBookQuery`
- `MirrorNodeContractCallQuery` (HIP-1027), `MirrorNodeContractEstimateGasQuery`
  — these belong on `mirrornode.contract.ContractRepository`, not in `consensusnode.queries`

### 1.11 Features on `Transaction<...>` / `PackedTransaction<...>`

The lifecycle (build → pack → sign → submit) is now explicit. `Submittable` (defined in
`consensusnode.client`) is the shared retry/submit base for both `PackedTransaction` and
`Query`.

| Feature in v2 | V3 spec |
| --- | --- |
| Explicit freeze step | :white_check_mark: via `Transaction.pack(payer, nodes)` |
| `addSignature(PublicKey, byte[])` (offline signing) | :white_check_mark: via `PackedTransaction.sign(list<NodeSignature>)` |
| `getSignatures()` / `signableNodeBodyBytesList()` (HSM workflow) | :white_check_mark: via `nodeSignatures` field + `signableBodies()` |
| `getTransactionHash()` / `getTransactionHashPerNode()` | :x: |
| `setRegenerateTransactionId(boolean)` | :x: |
| `batchify(Client, Key)` | :x: |
| Jumbo-tx size (HIP-1300): `bodySize`, `bodySizeAllChunks` | :x: |

---

## 2. `base` — missing types and concepts

### 2.1 Identifier types

Today only a generic `Address` exists. In v2 these are first-class types:

| Type | V3 spec |
| --- | --- |
| `AccountId` (with alias + EVM-address alias, HIP-583) | :warning: partially via `Address` |
| `ContractId` (incl. `fromEvmAddress`) | :x: |
| `DelegateContractId` | :x: |
| `TokenId` | :x: |
| `TopicId` | :x: |
| `FileId` | :x: |
| `ScheduleId` | :x: |
| `NftId` (`TokenId` + serial) | :x: |
| `PendingAirdropId` | :x: |
| `HookId`, `HookEntityId` | :x: |
| `EvmAddress` (as a type, not just a string) | :x: |
| `LedgerId` (MAINNET / TESTNET / PREVIEWNET / custom) | :warning: identifier constants, no type |
| `SemanticVersion` | :x: |
| `TransactionHash` (as a type, separate from `TransactionId`) | :x: |

### 2.2 Keys

| API in v2 | V3 spec |
| --- | --- |
| `KeyList` (n-of-n) | :x: |
| `ThresholdKey` (m-of-n) | :x: |
| `Mnemonic` (BIP-39 12 / 24-word + legacy 22-word, `toStandardEd25519PrivateKey`, `toStandardECDSAsecp256k1PrivateKey`) | :x: |
| `PublicKey.toEvmAddress()` / `toAccountId()` | :x: |
| HSM signer function (`UnaryFunction<byte[], byte[]>`) | :warning: abstracted via `TransactionSigner` |

### 2.3 Custom fees (HTS + HIP-991)

| Type | V3 spec |
| --- | --- |
| `CustomFee` (abstract) | :warning: read-only models on mirror node |
| `CustomFixedFee` | :warning: read model |
| `CustomFractionalFee` (`INCLUSIVE` / `EXCLUSIVE`, `netOfTransfers`) | :warning: read model |
| `CustomRoyaltyFee` (with `fallbackFee`) | :warning: read model |
| `CustomFeeLimit` (HIP-991, scheduled + topic submit) | :x: |
| `AssessedCustomFee` (in record) | :x: |
| Write-side custom-fee builder for `TokenCreate` / `TopicCreate` | :x: |

### 2.4 Fee schedule / fee estimation

`FeeSchedule`, `TransactionFeeSchedule`, `FeeComponents`, `FeeData`,
`FeeEstimate` / `FeeEstimateQuery` / `FeeEstimateResponse` (HIP-1313 with
high-volume multiplier) — **all :x:**

### 2.5 Records / receipts — fields on `Receipt` / `Record`

Today `Receipt` / `Record<$$Receipt>` only carries base fields. Missing v2
fields include: `exchangeRate`, `topicSequenceNumber`, `topicRunningHash`,
`totalSupply`, `serials`, `scheduleId`, `assessedCustomFees`,
`automaticTokenAssociations`, `paidStakingRewards`, `evmAddress`,
`prngBytes` / `prngNumber`, `signerNonce`, `pendingAirdropRecords`.

### 2.6 Status / enums

`BasicTransactionStatus` in V3 has ~20 values; the v2 `Status` enum has
**~300+ values** (`BUSY`, `THROTTLED_AT_CONSENSUS`, `INSUFFICIENT_PAYER_BALANCE`,
`INVALID_SIGNATURE`, ...). The open-extensible model is good, but the common
codes should ship in the standard mapping. Also missing: `TokenKeyValidation`
(HIP-540), `FreezeType`, `RequestType`.

---

## 3. `mirror-node-client` — missing endpoints

### 3.1 Entirely missing domains

| Domain | V3 spec |
| --- | --- |
| **Blocks** (HIP-415): `GET /blocks`, `GET /blocks/{hashOrNumber}` | :x: |
| **Schedules**: `GET /schedules`, `GET /schedules/{id}` (filters: `executed`, `account.id`, `schedule.id`) | :x: |

### 3.2 Missing endpoints per existing domain

**Accounts** — missing:

- `GET /accounts/{id}/rewards`
- `GET /accounts/{id}/allowances/{crypto|tokens|nfts}`
- `GET /accounts/{id}/airdrops/{outstanding|pending}` (HIP-904)
- `GET /balances` (top-level balance query)

**Contracts** — missing:

- `GET /contracts/{id}/results`, `/results/{txIdOrHash}`,
  `/results/{timestamp}`, `/results/{txIdOrHash}/actions`
- `GET /contracts/{id}/state`
- `GET /contracts/{id}/results/logs`, `/results/opcodes`
- `GET /contracts/logs` (filter `topic0`..`topic3`)
- `POST /contracts/call` (HIP-584 / HIP-1027, `eth_call` equivalent)

**Network** — missing:

- `POST /network/fees` (fee estimation, HIP-1313)
- `GET /network/nodes` (address book — missing as a repository method)

**Tokens** — missing:

- `GET /tokens/{id}/nfts/{serial}/transactions` (NFT history)

**Transactions** — missing:

- `GET /transactions/{txId}/stateproof`

### 3.3 Query parameter / filter concepts

- **Range / operator filters** on `timestamp`, `balance`, `sequencenumber`
  (`eq:`, `gt:`, `gte:`, `lt:`, `lte:`, `ne:` plus double-parameter ranges) —
  no generic model in the spec
- **Cursor-based pagination** via `links.next` — `Page<$$T>` exists, but no
  token / cursor concept is documented
- `order` (ASC / DESC), `limit` (max 100) — missing throughout

---

## 4. `enterprise` service layer — gaps to full coverage

| Service / method | V3 spec |
| --- | --- |
| **ScheduleService** (create / sign / delete + long-term) | :x: |
| **AllowanceService** (HBAR / token / NFT approve + delete) | :x: |
| **StakingService** (stake to node / account, decline reward, claim rewards) | :x: |
| **HCS**: `updateTopic()`, `subscribeWithFilter()`, topic custom fees (HIP-991) | :x: |
| **TokenService**: custom fees, pause / unpause, freeze / unfreeze, KYC, wipe, fee-schedule update, NFT updates, airdrop / claim / cancel / reject | :x: |
| **AccountService**: `transferHbar()`, alias / EVM-address, auto account create via transfer | :x: |
| **SmartContractService**: update / delete, Ethereum transactions, mirror-node-based `eth_call`, full param encoding (all Solidity types + arrays) | :warning: partial — only primitive params |
| **BatchService** (HIP-551 atomic batch across service calls) | :x: |
| **HooksService** (HIP-1195) | :x: |

Also still open in **Questions & Comments**:

- Cross-process consistency-token propagation
  (`exportConsistencyToken` / `importConsistencyToken`) in
  `enterprise.service.Session`
- Explicit per-call timeout vs. session-level configuration
- Dedicated `consistency-timeout-error` vs. generic `service-error`
- `Ledger` ↔ `Network` naming
- `ExchangeRate.exchangeRateInUsdCents`: `double` vs. `decimal`
- Meaning of `denominatingTokenId` in the topic / token mirror schemas

---

## 5. Client / networking / operator setup (`HieroClient`)

### 5.1 Operator identity is coupled to an in-memory `PrivateKey`

`HieroClient` requires an `operatorAccount: Account`
([`spec/consensus-node-client/client.md`](spec/consensus-node-client/client.md)), and
`Account` itself carries an `@@immutable privateKey: PrivateKey`. Constructing a client
therefore implies the operator's private key sits in process memory.

That assumption breaks for the majority of enterprise deployments:

- HSM / KMS-backed operator keys (AWS CloudHSM, Azure Key Vault, GCP KMS, PKCS#11)
- Hardware-wallet-backed operators (Ledger, Trezor, Blade-Signer)
- Browser-/Wallet-bridge integrations (HashConnect, WalletConnect, Hedera Wallet Snap)
- FIPS-certified custody where keys must never leave the device

v2 already has this escape hatch: `Client.setOperatorWith(AccountId, PublicKey,
UnaryFunction<byte[], byte[]>)` decouples operator identity (`AccountId` + `PublicKey`)
from the signing mechanism (a function pointer that an HSM / wallet wrapper provides).

V3 has the right building block in place — `TransactionSigner` on `HieroClient` — but
still forces `Account.privateKey` to exist alongside it. The fix is to **separate operator
identity from operator signing** at the type level:

| What | Current | Needed |
| --- | --- | --- |
| Operator identity | `Account { accountId, privateKey }` | `accountId: Address` + `publicKey: PublicKey` (no `PrivateKey`) |
| Operator signing | `transactionSigner: TransactionSigner` (already present) | unchanged; becomes the *only* signing path |
| In-memory shortcut | implicit via `Account.privateKey` | dedicated factory that wraps a `PrivateKey` in a `TransactionSigner` for CLI / dev use |

This change has ripple effects:

- `Account` either loses `privateKey` (becoming an identity-only record) or is split into
  `Account` (identity) and `LocalAccount`/`SigningAccount` (identity + key).
- `Transaction.sign(payer: Account, nodes)` and `PaidQuery`'s sponsored-query story (see
  [ADR-0002](docs/adr/0002-defer-paid-query-payer-customization.md)) both feed from the
  same operator/payer-identity model and must be reworked consistently.
- The 95-% CLI / script case should still be one line — a static helper to wrap a
  `PrivateKey` into the new identity + signer pair keeps the simple path simple.

This is an architectural correction, not a feature addition. It should land before V3
attempts to address sponsored-query payers (which depend on the same identity / signer
decoupling) or shipping consumers for whom external signing is non-negotiable.

### 5.2 Other client features missing from v2

The V3 spec has a lean `HieroClient<$$Unit>`. v2 offers significantly more
setup flexibility:

| Feature in v2 | V3 spec |
| --- | --- |
| `forMainnet` / `forTestnet` / `forPreviewnet` / `forName` / `forNetwork` / `forMirrorNetwork` (HIP-1064) / `fromConfig` factory variants | :warning: abstracted via `NetworkSetting` registry |
| `setMaxQueryPayment`, `setDefaultMaxTransactionFee` | :x: |
| `setRequestTimeout`, `setGrpcDeadline` | :warning: partially at the transaction level |
| `setMinBackoff` / `setMaxBackoff` / `setMaxAttempts` as client defaults | :warning: only tx-level |
| `setNetworkUpdatePeriod` (automatic address-book refresh) | :x: |
| `setLogger(Logger)` (pluggable logger) | :x: |
| Persistent shard / realm (HIP-1299) | :x: |
| `setNetworkFromAddressBook(NodeAddressBook)` | :x: |

---

## 6. Sub-layers without content

- `spec/consensus-node-client/proto.md` and `proto-accounts.md` — empty
  placeholders
- `spec/base/proto.md`, `grpc.md` — minimal (only `MethodDescriptor`); too
  thin for a multi-service SDK with SPI support (custom services)
- `consensusnode.transactions.spi.TransactionSupport` is well specified but
  has **no second example implementation** beyond accounts — the SPI's
  load-bearing capacity has not been validated through a second / third
  service yet

---

## 7. Suggested prioritization

1. **SDK parity (must-have):** `TransferTransaction` (HBAR), token-service
   transactions (`Create` / `Associate` / `Mint` / `Burn` / transfer),
   topic transactions (`Create` / `Update` / `Delete` / `SubmitMessage`),
   file transactions, contract transactions + `EthereumTransaction`, the
   remaining common queries (`AccountRecordsQuery`, contract / file / token /
   topic / schedule info queries). Query base abstractions + first two
   concrete queries already landed; pattern is established for the rest.
2. **Modern HIPs:** Schedule (HIP-755), airdrops (HIP-904), DAB node ops
   (HIP-869 / HIP-1046), custom fees on topics (HIP-991), batch (HIP-551),
   mutable NFT metadata (HIP-657).
3. **Auxiliary:** PRNG, `Freeze` / `SystemDelete`, allowances, staking
   transactions, hooks (HIP-1195), `Mnemonic` + `KeyList` / `ThresholdKey`,
   full custom-fee hierarchy for builders.
4. **Mirror node:** blocks, schedules, contracts
   (results / state / logs / call), airdrops, allowances, rewards, generic
   range-filter model.
5. **Client / infrastructure:** fee estimation, logger abstraction,
   address-book refresh, shard / realm persistence.

---

## Sources

- [Hiero Java SDK](https://github.com/hiero-ledger/hiero-sdk-java)
- [Hiero Java SDK CHANGELOG](https://github.com/hiero-ledger/hiero-sdk-java/blob/main/CHANGELOG.md)
- [Hedera SDK transactions overview](https://docs.hedera.com/hedera/sdks-and-apis/sdks/transactions)
- [Hedera Mirror Node REST API](https://docs.hedera.com/hedera/sdks-and-apis/rest-api)
- [Mirror Node Swagger (Mainnet)](https://mainnet-public.mirrornode.hedera.com/api/v1/docs/)
- [HIP-904 Frictionless Airdrops](https://hips.hedera.com/hip/hip-904)
- [HIP-991 Permissionless Revenue-Generating Topics](https://hips.hedera.com/hip/hip-991)
- [HIP-1195 Hiero Hooks](https://hips.hedera.com/hip/hip-1195)
- [HIP-415 Blocks](https://hips.hedera.com/HIP/hip-415.html)
- [Hedera client setup guide](https://docs.hedera.com/hedera/sdks-and-apis/sdks/client)
