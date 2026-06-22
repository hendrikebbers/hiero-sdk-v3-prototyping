# Token Queries API

This namespace defines the read-side counterpart to the token transactions in
[`transactions-tokens.md`](transactions-tokens.md). Both queries are *paid*: the consensus node
charges a fee for serving token metadata.

## Description

Two queries are exposed:

- **`TokenInfoQuery`** extends `PaidQuery<TokenInfo>` — returns the full metadata snapshot of a
  token (HTS): name, symbol, supply, treasury, the eight optional keys, freeze / KYC / pause
  status, expiration, auto-renew configuration, and the `deleted` flag. Works for both
  `FUNGIBLE_COMMON` and `NON_FUNGIBLE_UNIQUE` tokens — the `type` field distinguishes them.

- **`TokenNftInfoQuery`** extends `PaidQuery<TokenNftInfo>` — returns the metadata of a single
  NFT, identified by its token id **and** serial number: current owner, creation time, the
  opaque per-serial `metadata`, and the approved spender (if any). Only meaningful for
  `NON_FUNGIBLE_UNIQUE` tokens.

Both extend `PaidQuery`, so they inherit the `maxQueryPayment` knob, the `getCost(client)`
cost-discovery method, and the `PaidQueryResponse<...>` envelope. See
[`queries.md`](queries.md) for the full payment / envelope semantics.

A query against a deleted token does not fail at the protocol level — `TokenInfoQuery` returns a
`TokenInfo` with `deleted = true` and the remaining fields reflecting the token's last state.
Callers should branch on `deleted` rather than assume the snapshot is live. A query against a
token id that never existed, or a serial that was never minted (or has been burnt), fails at the
protocol level rather than returning a tombstone.

### Tri-state freeze / KYC / pause status

HAPI returns the freeze, KYC, and pause status of a token as three-valued enums whose third value
means "the token has no such key, so the concept does not apply" (`FreezeNotApplicable`,
`KycNotApplicable`, `PauseNotApplicable`). V3 collapses the "not applicable" value into `null`:
a `@@nullable bool` where `null` means the corresponding key is unset (concept disabled),
`true`/`false` carry the actual default / current status. This keeps the result honest without
introducing three near-identical enums — see *Questions & Comments*.

## API Schema

```
namespace consensusnode.queries.tokens
requires {Address, AccountId} from ledger
requires {PublicKey} from keys
requires {TokenType, TokenSupplyType} from token
requires {PaidQuery} from consensusnode.queries

// Full token metadata snapshot. Returned by TokenInfoQuery. Covers both FUNGIBLE_COMMON and
// NON_FUNGIBLE_UNIQUE tokens; `type` distinguishes them. A deleted token is returned with
// `deleted = true` rather than failing the query.
type TokenInfo {
    @@immutable tokenId: Address
    @@immutable type: TokenType                                 // FUNGIBLE_COMMON | NON_FUNGIBLE_UNIQUE
    @@immutable @@maxLength(100) name: string
    @@immutable @@maxLength(100) symbol: string
    @@immutable @@default(0) decimals: int32                    // smallest-unit precision; 0 for NON_FUNGIBLE_UNIQUE
    @@immutable totalSupply: int64                              // current total supply, in the smallest indivisible unit
    @@immutable treasuryAccountId: AccountId                    // account holding the supply pool
    @@immutable supplyType: TokenSupplyType                     // INFINITE | FINITE
    @@immutable @@nullable maxSupply: int64                     // FINITE only: protocol-enforced ceiling; null when supplyType is INFINITE
    @@immutable @@nullable tokenMemo: string                    // short human-readable label
    @@immutable @@default([]) metadata: bytes                   // opaque token-level metadata (e.g. IPFS CID, HTTPS URL, JSON manifest)
    @@immutable expirationTime: zonedDateTime
    @@immutable @@nullable autoRenewPeriod: seconds
    @@immutable @@nullable autoRenewAccount: AccountId          // account paying for auto-renewal; null → the token itself pays
    @@immutable @@default(false) deleted: bool                  // true if the token has been deleted (other fields reflect its last state)

    // The eight optional keys (see transactions-tokens.md "Token keys"). A null key means the
    // capability was never granted at create and can never be added.
    @@immutable @@nullable adminKey: PublicKey                  // controls update / delete; null → immutable token
    @@immutable @@nullable supplyKey: PublicKey                 // controls mint / burn; null → fixed supply
    @@immutable @@nullable kycKey: PublicKey                    // controls KYC grant / revoke; null → KYC disabled
    @@immutable @@nullable freezeKey: PublicKey                 // controls freeze / unfreeze; null → no freeze
    @@immutable @@nullable wipeKey: PublicKey                   // controls wipe; null → wipe disabled
    @@immutable @@nullable pauseKey: PublicKey                  // controls pause / unpause; null → unpausable
    @@immutable @@nullable feeScheduleKey: PublicKey            // controls custom-fee schedule updates; null → fees immutable
    @@immutable @@nullable metadataKey: PublicKey               // controls per-serial NFT metadata updates (HIP-657); null → metadata immutable

    // Tri-state status collapsed to nullable bool: null → corresponding key unset (not applicable).
    @@immutable @@nullable defaultFreezeStatus: bool            // true → newly associated accounts start FROZEN; null → no freezeKey
    @@immutable @@nullable defaultKycStatus: bool               // true → newly associated accounts start KYC-granted; null → no kycKey
    @@immutable @@nullable paused: bool                         // true → token is currently paused; null → no pauseKey
}

// Paid query for the metadata snapshot of a token (fungible or NFT collection). Does not return
// per-serial NFT data — use TokenNftInfoQuery for a single serial.
@@finalType
TokenInfoQuery extends PaidQuery<TokenInfo> {
    @@immutable tokenId: Address
}

// Metadata of a single NFT serial. Returned by TokenNftInfoQuery. Only meaningful for
// NON_FUNGIBLE_UNIQUE tokens.
type TokenNftInfo {
    @@immutable tokenId: Address                                // the NFT collection
    @@immutable serial: int64                                   // serial number within the collection
    @@immutable accountId: AccountId                            // current owner of this serial
    @@immutable creationTime: zonedDateTime                     // consensus time at which this serial was minted
    @@immutable @@default([]) metadata: bytes                   // opaque per-serial metadata set at mint (e.g. IPFS CID)
    @@immutable @@nullable spenderId: AccountId                 // account granted an allowance to transfer this serial (HIP-376); null if none
}

// Paid query for a single NFT serial, identified by its collection `tokenId` and `serial`.
@@finalType
TokenNftInfoQuery extends PaidQuery<TokenNftInfo> {
    @@immutable tokenId: Address
    @@immutable @@min(1) serial: int64
}
```

## Examples

### Read token metadata, branch on deleted (paid)

```
HieroClient client = ...;

TokenInfo info = new TokenInfoQuery()
    .tokenId(tokenId)
    .submit(client)
    .value;

if (info.deleted) {
    return;                                     // token no longer usable; treat remaining fields as historical
}

int64     supply  = info.totalSupply;
PublicKey admin   = info.adminKey;              // null → immutable token
PublicKey supply2 = info.supplyKey;             // null → fixed supply
```

### Distinguish fungible from NFT

```
TokenInfo info = new TokenInfoQuery().tokenId(tokenId).submit(client).value;

if (info.type == TokenType.NON_FUNGIBLE_UNIQUE) {
    // info.decimals is 0; info.totalSupply is the number of minted serials
} else {
    // FUNGIBLE_COMMON: info.totalSupply is in the smallest unit (apply info.decimals to display)
}
```

### Read a single NFT serial (paid)

```
TokenNftInfo nft = new TokenNftInfoQuery()
    .tokenId(collectionId)
    .serial(42L)
    .submit(client)
    .value;

AccountId owner   = nft.accountId;
bytes     meta    = nft.metadata;
AccountId spender = nft.spenderId;              // null if no allowance is set
```

### Confirm price up-front

```
PaidQuery<TokenInfo> query = new TokenInfoQuery().tokenId(tokenId);

NativeToken<ANY, ANY> quotedCost = query.getCost(client);
// show quotedCost to the user, wait for approval, then:

TokenInfo info = query.submit(client).value;
```

`getCost(client)` is non-binding — the price on the subsequent `submit(client)` may differ;
combine it with `maxQueryPayment` for a hard ceiling. See
[`queries.md`](queries.md#examples) for the full pattern.

## Questions & Comments

- **All key fields are typed as `PublicKey`, not `Key`.** Same placeholder typing as on the
  write side; see [`transactions-tokens.md`](transactions-tokens.md) *Questions & Comments* for
  why a single `PublicKey` cannot express the recursive `Key` sum type that HAPI actually returns
  (entries may be `KeyList`, `ThresholdKey`, `ContractID`, …). Tracked in
  [`missing-features.md`](../../missing-features.md) section 3.2 — the `Key` row is a
  prerequisite for retyping all eight key fields here.

- **`tokenId` / `nftId` are the generic `Address` placeholder.** Token / NFT ids are not yet
  promoted to typed `TokenId` / `NftId` (token id + serial) identifiers — tracked in
  [`missing-features.md`](../../missing-features.md) section 3.1, with
  [`token.md`](../base/token.md) the intended landing place. `TokenNftInfoQuery` therefore takes
  `tokenId` + `serial` as two fields rather than a single `NftId`; once `NftId` lands, the input
  should collapse to one field (breaking change for the query input, additive for the result).

- **Tri-state freeze / KYC / pause status collapsed to `@@nullable bool`.** HAPI models each as a
  three-valued enum (`*NotApplicable` / on / off). The "not applicable" value is fully implied by
  the corresponding key being unset, so the spec drops it: `null` ⇔ key unset, `true`/`false`
  ⇔ the actual status. If the redundancy of carrying both the null key *and* a "not applicable"
  status ever proves useful (e.g. for callers that do not also read the key fields), three small
  enums in `base/token` would be the additive replacement.

- **No `customFees` on `TokenInfo`.** HAPI's `TokenGetInfo` carries the token's custom-fee
  schedule (`list<CustomFee>`), but the write-side `CustomFee` hierarchy is not yet specified for
  the consensus-node layer (see [`transactions-tokens.md`](transactions-tokens.md) *Questions &
  Comments* and [`missing-features.md`](../../missing-features.md) section 3.3). Read-side
  custom-fee shapes already exist on the mirror-node side
  ([`mirror-node-token.md`](../mirror-node-client/mirror-node-token.md)). Adding
  `customFees: list<CustomFee>` here once the write-side model lands is additive.

- **No `ledgerId` on `TokenInfo` / `TokenNftInfo`.** Matches the choice made for `AccountInfo`,
  `FileInfo`, and `TopicInfo` — the caller already knows which ledger they queried through their
  `HieroClient`, and ledger-origin metadata belongs on the response envelope, not duplicated on
  every typed payload. The corresponding envelope-level question is opened in
  [`queries.md`](queries.md) *Questions & Comments*.

- **`TokenInfo` here is distinct from `mirrornode.token.TokenInfo`.** Same conceptual data but
  different sources (consensus node gRPC vs. mirror node REST) and slightly different field
  semantics (e.g. the mirror-node `TokenInfo` carries `createdTimestamp` / `modifiedTimestamp`
  and an embedded `customFees`, while this consensus-node snapshot is point-in-time and omits
  custom fees for now). The two are kept separate to avoid coupling the consensus-node API to
  mirror-node update cadence — same split as `AccountInfo` vs. `mirrornode.account.AccountInfo`.
  Likewise `TokenNftInfo` here is distinct from `mirrornode.nft.Nft`.
