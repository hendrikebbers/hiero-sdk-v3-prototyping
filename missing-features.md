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

### 1.2 Token service (HTS)

Still open:

- `TokenFeeScheduleUpdate` — sole payload is the custom-fee schedule; blocked on the write-side
  `CustomFee` hierarchy (§3.3).
- HIP-540 key-clearing — deferred to a future write-side `KeyUpdate` type
  ([ADR-0004](docs/adr/0004-authority-authorization-sum-type.md)).
- `TokenCreate.customFees` write-side builder — depends on §3.3.
- Airdrop *pending-airdrop* record field — part of the base-`Record` design decision (§3.5).

### 1.3 File service

Open: chunked-append model (per-chunk receipts vs. single-receipt facade) — tracked in
[`transactions-files.md`](spec/consensus-node-client/transactions-files.md) *Questions & Comments*.

### 1.4 Smart contract service

`ContractCreate`, `ContractCreateFlow`, `ContractUpdate`, `ContractExecute`,
`ContractDelete`, `EthereumTransaction` — **all :x:**

### 1.5 Topic / HCS

`TopicMessageSubmit` (chunked) — :x: (shares the chunking design question with
`FileAppend`). HIP-991 custom fees on topics (`customFees`, `feeScheduleAuthority`,
`feeExemptAuthorities`) — :x: (depends on write-side custom-fee model from §3.3).

### 1.6 Schedule service (HIP-755 long-term)

`ScheduleCreate`, `ScheduleSign`, `ScheduleDelete` + `customFeeLimits` —
**all :x:**

### 1.7 Utility

- `PrngTransaction` (HIP-351) — :x:
- `BatchTransaction` (HIP-551, with `batchify()` on inner transactions) — :x:
- Hiero Hooks (HIP-1195): `HookStoreTransaction`, `EvmHook`, `EvmHookCall`,
  `HookCreationDetails`, ... — :x:

### 1.8 Queries (consensus-node-side)

The base abstractions (`Query` / `PaidQuery` / `QueryResponse` / `PaidQueryResponse`) and the
account / file / token / topic info queries are specified. Still missing:

- `ContractCallQuery` (local EVM call), `ContractInfoQuery`, `ContractBytecodeQuery`
- `ScheduleInfoQuery`
- `MirrorNodeContractCallQuery` (HIP-1027), `MirrorNodeContractEstimateGasQuery`
  — these belong on `mirrornode.contract.ContractRepository`, not in `consensusnode.queries`

### 1.9 Features on `Transaction<...>` / `PackedTransaction<...>`

The lifecycle (build → pack → sign → submit) is specified, including the offline / HSM signing
surface (`signableBodies()`, `sign(list<NodeSignature>)`). Still missing:

| Feature in v2 | V3 spec |
| --- | --- |
| `getTransactionHash()` / `getTransactionHashPerNode()` | :x: deliberately deferred — see note below |
| `setRegenerateTransactionId(boolean)` | :x: will NOT land in V3 — see note below (and [`transactions.md`](spec/consensus-node-client/transactions.md) *Questions & Comments*) |
| `batchify(Client, Key)` | :x: |
| Jumbo-tx size (HIP-1300): `bodySize`, `bodySizeAllChunks` | :x: |

Notes on the two annotated rows:

- **`getTransactionHash()` / `getTransactionHashPerNode()` — deliberately deferred.** These are
  pure SDK conveniences that expose the standard Hedera on-chain transaction hash (SHA-384 of
  the `SignedTransaction` bytes — and one hash per target node, because the body's
  `nodeAccountID` field differs per node). They are *not* required for any V3 capability: the
  hash is reconstructible by any caller from `PackedTransaction.toBytes()` (single submitted
  node) or from `PackedTransaction.signableBodies()` plus the collected `NodeSignature` set
  (per-node hashes). Mirror-node correlation by hash (`GET /api/v1/transactions/{hash}`) is
  the most common driver for exposing these on `PackedTransaction`; we will revisit once a
  concrete consumer in `mirrornode.*` or `enterprise.service.*` needs them. Until then, keeping
  the surface narrow is preferred — adding `transactionHash` / `nodeTransactionHashes`
  accessors later is purely additive.

- **`setRegenerateTransactionId(boolean)` — will NOT land in V3.** Architecturally incompatible
  with the V3 transaction model: each `NodeSignature` signs the serialized `TransactionBody`,
  which carries the `TransactionId`; regenerating the id changes the body bytes and silently
  invalidates every signature already collected. The V3 model is built around the
  byte-stability boundary that the `Transaction` → `PackedTransaction` split makes visible
  (ADR-0001), and the offline / multi-party / HSM signing flows (`signableBodies()`,
  `sign(list<NodeSignature>)`, `fromBytes(...)`) all depend on it. A "regenerate and retry"
  toggle would either silently drop already-collected signatures (data loss the custodians did
  not consent to) or break the contract that `toBytes()` is the canonical payload. The V3
  answer to `TRANSACTION_EXPIRED` is therefore explicit: the caller re-builds
  (`Transaction.pack(...)`) and re-collects signatures. A future single-signer convenience
  ("pack + sign-with-operator + submit; on expiry, retry the whole cycle") may live in the
  `enterprise.service.*` layer where loss-of-signatures is moot. Full rationale in
  [`transactions.md`](spec/consensus-node-client/transactions.md) *Questions & Comments*.

---

## 2. `consensus-node-admin-client` — admin-only transactions

The admin module (privileged freeze / system-delete / Dynamic Address Book operations, kept out of
the surface normal apps consume; depends on `consensus-node-client` but is not imported by it) is
specified. Remaining open items, each tracked in its file's *Questions & Comments*:

- **System / network admin** ([`transactions-freeze.md`](spec/consensus-node-admin-client/transactions-freeze.md),
  [`transactions-system.md`](spec/consensus-node-admin-client/transactions-system.md)): typed
  `NetworkAdminKey` reference; splitting the `freezeType` / `@@oneOf(fileId, contractId)` payloads
  into concrete subtypes.
- **Dynamic Address Book** ([`transactions-nodes.md`](spec/consensus-node-admin-client/transactions-nodes.md)):
  typed `NodeId` / `X509Certificate` placeholders. (Typed `IpAddress` has landed in `base/ledger.md`.)
- **Admin-side queries** ([`queries-network.md`](spec/consensus-node-admin-client/queries-network.md)):
  extract `SemanticVersion` into `base/common` once §3.1 lands a base type; surface tombstoned
  node-address entries explicitly.

---

## 3. `base` — missing types and concepts

### 3.1 Identifier types

The address hierarchy (`BaseAddress` → `EvmCapableAddress` → `Address` / `ContractId` /
`AccountId`, plus the standalone `EvmAddress`) has **landed** in
[`base/ledger.md`](spec/base/ledger.md), together with the call-site migration to these typed
identifiers and the `ZERO_ADDRESS` / `ZERO_ACCOUNT_ID` / `ZERO_CONTRACT_ID` clear-sentinels (see
[ADR-0003](docs/adr/0003-three-level-address-hierarchy-with-nullability-narrowing.md)). Some
`mirror-node-contract.md` fields (`obtainerId`, `proxyAccountId`, `fileId`) still use raw `string`
and carry `// TODO` markers for migration to `AccountId` / `Address`.

#### Composite identifier types — still missing as **types**

These are not single-selector addresses but composites of multiple
`BaseAddress`-typed fields plus extra information. They land alongside
the feature that first needs each one:

| Type | Composition | Lands with |
| --- | --- | --- |
| `NftId` | `(tokenId: Address, serial: int64)`; currently spread across two parameters in `NftTransfer` and `TokenBurnTransaction.serials` | NFT-touching APIs that take a single-NFT argument — HIP-657 `TokenUpdateNfts`, HIP-904 airdrops |
| `PendingAirdropId` | `(senderId: AccountId, receiverId: AccountId, tokenId: Address[, serial: int64])`; 3- or 4-tuple depending on fungible vs NFT | HIP-904 `TokenAirdrop` / `TokenClaimAirdrop` / `TokenCancelAirdrop` (§1.2) |
| `HookId` / `HookEntityId` | composite (entity-bound) per HIP-1195 | Hiero Hooks (§1.7) |

#### Other identifier-shaped types tracked here

Not entity ids in the `(shard, realm, X)` sense, but still genuinely
missing as typed values:

| Type | Why it is a type and not a constant | V3 spec |
| --- | --- | --- |
| `LedgerId` (MAINNET / TESTNET / PREVIEWNET / custom) | a value object identifying which network a transaction / address belongs to; today exposed only as identifier constants in [`base/ledger.md`](spec/base/ledger.md) | :warning: constants only, no type |
| `SemanticVersion` | `(major, minor, patch, pre, build)` value object used by `NetworkVersionInfoQuery` ([`queries-network.md`](spec/consensus-node-admin-client/queries-network.md)) | :x: |
| `TransactionHash` | SHA-384 of the `SignedTransaction` bytes; distinct from `TransactionId` (the payer + valid-start tuple), and the actual lookup key on the mirror node (`GET /api/v1/transactions/{hash}`) | :x: (gated behind §1.9's deferred `getTransactionHash*` decision — if those accessors don't land, the typed `TransactionHash` may not be needed either) |

### 3.2 Keys

The HAPI authorization-key model has landed as the [`authority`](spec/base/authority.md) namespace
(`Authority` sum type — see [ADR-0004](docs/adr/0004-authority-authorization-sum-type.md)); all key
fields across the specs are now typed `Authority`. Two follow-ups remain open: `@@sealed` must be
added to the meta-language (the spec uses it ahead of that), and key *clearing* (HIP-540) is
deferred to a future write-side `KeyUpdate` type. Still missing:

| API in v2 | V3 spec |
| --- | --- |
| `Mnemonic` (BIP-39 12 / 24-word + legacy 22-word, `toStandardEd25519PrivateKey`, `toStandardECDSAsecp256k1PrivateKey`) | :x: |
| `PublicKey.toEvmAddress()` / `toAccountId()` | :x: |
| HSM signer function (`UnaryFunction<byte[], byte[]>`) | :warning: abstracted via `TransactionSigner` |

### 3.3 Custom fees (HTS + HIP-991)

| Type | V3 spec |
| --- | --- |
| `CustomFee` (abstract) | :warning: read-only models on mirror node |
| `CustomFixedFee` | :warning: read model |
| `CustomFractionalFee` (`INCLUSIVE` / `EXCLUSIVE`, `netOfTransfers`) | :warning: read model |
| `CustomRoyaltyFee` (with `fallbackFee`) | :warning: read model |
| `CustomFeeLimit` (HIP-991, scheduled + topic submit) | :x: |
| `AssessedCustomFee` (in record) | :x: |
| Write-side custom-fee builder for `TokenCreate` / `TopicCreate` | :x: |

### 3.4 Fee schedule / fee estimation

`FeeSchedule`, `TransactionFeeSchedule`, `FeeComponents`, `FeeData`,
`FeeEstimate` / `FeeEstimateQuery` / `FeeEstimateResponse` (HIP-1313 with
high-volume multiplier) — **all :x:**

### 3.5 Records / receipts — fields on `Receipt` / `Record`

Today the V3 base types carry:

- `Receipt`: `transactionId`, `status`, `exchangeRate`, `nextExchangeRate`
- `Record<$$Receipt>`: `transactionId`, `consensusTimestamp`, `receipt`

(`exchangeRate` / `nextExchangeRate` are already on the base, so they are
**not** in the missing list below.) The remaining v2 fields split into two
distinct groups: transaction-specific Receipt fields, which collapse cleanly
into the existing typed `XxxReceipt extends Receipt` pattern; and Record
fields, which need a base-`Record` design decision before they can land
cleanly.

#### Receipt-level fields (transaction-specific)

| Field(s) | Set by | V3 home |
| --- | --- | --- |
| `topicSequenceNumber` + `topicRunningHash` (+ `topicRunningHashVersion`) | `TopicMessageSubmit` | `TopicMessageSubmitReceipt` (file pending — §1.5) |
| `totalSupply` | `TokenMint`, `TokenBurn`, `TokenWipe` | `TokenMintReceipt` :white_check_mark: / `TokenBurnReceipt` :white_check_mark: ([`transactions-tokens.md`](spec/consensus-node-client/transactions-tokens.md)); add to `TokenWipeReceipt` when it lands |
| `serials` | `TokenMint` (NFT only) | `TokenMintReceipt` :white_check_mark: |
| `scheduleId` (+ `scheduledTransactionId`) | `ScheduleCreate` | `ScheduleCreateReceipt` (file pending — §1.6) |
| `evmAddress` | `AccountCreate` (alias / auto-create path) **and** `ContractCreate` | `AccountCreateReceipt` (nullable — only populated on the alias path) **and** `ContractCreateReceipt` (file pending — §1.4) |
| `prngBytes` **xor** `prngNumber` | `UtilPrng` (HIP-351) — one of the two depending on the `range` parameter | `PrngReceipt`, modelled as `@@oneOf(prngBytes, prngNumber)` (file pending — §1.7) |

These seven fields are routine: each rides on the existing
typed-receipt-per-transaction pattern. Five of them are still pending only
because the transaction itself is still pending (`TopicMessageSubmit`,
`ScheduleCreate`, `UtilPrng`, plus the relevant fields on `AccountCreate` /
`ContractCreate` receipts); add the field at the same time the receipt is
specified. `totalSupply` + `serials` on the Token-Mint/Burn receipts are
already in.

#### Record-level fields (cross-cutting OR on a typed Record)

| Field | When set | V3 home |
| --- | --- | --- |
| `assessedCustomFees` | any transaction touching tokens-with-custom-fees — typically `TransferTransaction`, `TokenAirdrop` | base `Record<$$Receipt>` as `@@default([]) list<AssessedCustomFee>`; depends on the write-side `CustomFee` hierarchy in §3.3 |
| `automaticTokenAssociations` | any transaction whose execution auto-associates an account (`maxAutomaticTokenAssociations > 0`, first inbound transfer) | base `Record<$$Receipt>` as `@@default([]) list<TokenAssociation>` |
| `paidStakingRewards` | any transaction whose execution triggers a staking-reward payout to a participating account | base `Record<$$Receipt>` as `@@default([]) list<AccountAmount>` |
| `signerNonce` | `EthereumTransaction` only | typed `EthereumTransactionRecord` — not on the base |
| `pendingAirdropRecords` | `TokenAirdrop` only (HIP-904), when the recipient is not auto-associated | typed `TokenAirdropRecord` — not on the base |

The first three are **genuinely cross-cutting**: they can appear on most
transactions and depend on what happened at execution, not on the
transaction's declared type. The natural home is `Record<$$Receipt>` base,
each as `@@default([])` so consumers always see a non-null but
usually-empty list.

The last two are **strictly transaction-specific but live on the Record,
not the Receipt** (HAPI: `TransactionRecord.ethereum_nonce` /
`new_pending_airdrops` are record fields, not receipt fields). V3 today
does not type the record per transaction — `Record<$$Receipt>` carries
`receipt: $$Receipt` and is otherwise generic. To express `signerNonce` /
`pendingAirdropRecords` as type-specific record fields, `Record` would have
to be parameterised over a second generic (e.g.
`Record<$$Receipt, $$Extras>` or a `$$Record` sibling). That is an
architectural change that should land **before** either `EthereumTransaction`
or `TokenAirdrop` is specified, so that those specs can use the typed
record from the start rather than being retrofitted.

**Open architectural question** (worth raising in a Q&C entry on
[`transactions.md`](spec/consensus-node-client/transactions.md) before
the next record-touching transaction lands): should `Record<$$Receipt>`
carry a fixed set of cross-cutting fields (`assessedCustomFees`,
`automaticTokenAssociations`, `paidStakingRewards`), OR additionally be
parameterised over a `$$Record` (or `$$Extras`) type so that transactions
with type-specific record content (`signerNonce`,
`pendingAirdropRecords`, ...) can express it through the type system?
Both paths are compatible; the question is whether the typed-record
parameter is worth its complexity for the two known consumers, or whether
those two should live as `@@nullable` cross-cutting fields on the base
instead.

### 3.6 Status / enums

`BasicTransactionStatus` in V3 has ~20 values; the v2 `Status` enum has
**~300+ values** (`BUSY`, `THROTTLED_AT_CONSENSUS`, `INSUFFICIENT_PAYER_BALANCE`,
`INVALID_SIGNATURE`, ...). The open-extensible model is good, but the common
codes should ship in the standard mapping. Also missing: `TokenKeyValidation`
(HIP-540), `FreezeType`, `RequestType`.

---

## 4. `mirror-node-client` — missing endpoints

### 4.1 Entirely missing domains

| Domain | V3 spec |
| --- | --- |
| **Blocks** (HIP-415): `GET /blocks`, `GET /blocks/{hashOrNumber}` | :x: |
| **Schedules**: `GET /schedules`, `GET /schedules/{id}` (filters: `executed`, `account.id`, `schedule.id`) | :x: |

### 4.2 Missing endpoints per existing domain

**Contracts** — missing:

- `GET /contracts/{id}/results`, `/results/{txIdOrHash}`,
  `/results/{timestamp}`, `/results/{txIdOrHash}/actions`
- `GET /contracts/{id}/state`
- `GET /contracts/{id}/results/logs`, `/results/opcodes`
- `GET /contracts/logs` (filter `topic0`..`topic3`)
- `POST /contracts/call` (HIP-584 / HIP-1027, `eth_call` equivalent)

**Tokens** — missing:

- `GET /tokens/{id}/nfts/{serial}/transactions` (NFT history)

**Transactions** — missing:

- `GET /transactions/{txId}/stateproof`

### 4.3 Query parameter / filter concepts

- **Range / operator filters** on `timestamp`, `balance`, `sequencenumber`
  (`eq:`, `gt:`, `gte:`, `lt:`, `lte:`, `ne:` plus double-parameter ranges) —
  no generic model in the spec
- **Cursor-based pagination** via `links.next` — `Page<$$T>` exists, but no
  token / cursor concept is documented
- `order` (ASC / DESC), `limit` (max 100) — missing throughout

---

## 5. `enterprise` service layer — gaps to full coverage

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

## 6. Client / networking / operator setup (`HieroClient`)

### 6.1 Operator identity is coupled to an in-memory `PrivateKey`

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

### 6.2 Other client features missing from v2

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

## 7. Sub-layers without content

- `spec/consensus-node-client/proto.md` and `proto-accounts.md` — empty
  placeholders
- `spec/base/proto.md`, `grpc.md` — minimal (only `MethodDescriptor`); too
  thin for a multi-service SDK with SPI support (custom services)
- `consensusnode.transactions.spi.TransactionSupport` is well specified but
  has **no second example implementation** beyond accounts — the SPI's
  load-bearing capacity has not been validated through a second / third
  service yet

---

## 8. Suggested prioritization

1. **SDK parity (must-have):** the remaining write transactions — contract
   transactions + `EthereumTransaction`, `TopicMessageSubmit`, the token admin
   operations (wipe / freeze / KYC / pause / fee-schedule) — and the remaining
   common queries (contract / schedule info). The transaction lifecycle, transfers,
   token/topic/file core, and the account/file/token/topic info queries have landed,
   so the pattern is established for the rest.
2. **Modern HIPs:** Schedule (HIP-755), airdrops (HIP-904), DAB node ops
   (HIP-869 / HIP-1046 — land in `consensus-node-admin-client`, §2),
   custom fees on topics (HIP-991), batch (HIP-551),
   mutable NFT metadata (HIP-657).
3. **Auxiliary:** PRNG, `Freeze` / `SystemDelete` (also `consensus-node-admin-client`,
   §2), allowances, staking transactions, hooks (HIP-1195),
   `Mnemonic` + `KeyList` / `ThresholdKey`, full custom-fee hierarchy for builders.
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
