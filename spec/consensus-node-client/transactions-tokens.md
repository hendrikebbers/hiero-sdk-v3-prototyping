# Token Transactions API

## Description

The token service (HTS — Hedera Token Service, retained as the protocol-level name in Hiero) lets
users create, manage, and operate first-class tokens directly on the consensus node — without
deploying a smart contract. A single token is identified by an `Address` (token id) and is one of
two kinds:

- **`FUNGIBLE_COMMON`** — a divisible currency-like asset with a `decimals` precision and a
  shared supply pool. Balances are tracked per account as an `int64` count of the smallest
  indivisible unit.
- **`NON_FUNGIBLE_UNIQUE`** (NFT) — an indivisible collection where each issued unit is a
  distinct serial number with its own metadata. Balances are tracked per (token, serial).

A token's supply policy is independent of its kind:

- **`INFINITE`** — no protocol-enforced ceiling on `totalSupply`. The supply key may keep minting
  forever (modulo `int64` overflow).
- **`FINITE`** — the protocol enforces `totalSupply ≤ maxSupply` at every mint. A mint that would
  exceed `maxSupply` fails atomically.

This file covers the **core lifecycle and supply** transactions: `TokenCreate`, `TokenUpdate`,
`TokenDelete`, `TokenAssociate`, `TokenDissociate`, `TokenMint`, `TokenBurn`. The remaining
HTS transactions (`TokenWipe`, `TokenFreeze` / `TokenUnfreeze`, `TokenGrantKyc` / `TokenRevokeKyc`,
`TokenPause` / `TokenUnpause`, `TokenFeeScheduleUpdate`, `TokenUpdateNfts`, and the
HIP-904 airdrop family) are tracked in
[`missing-features.md`](../../missing-features.md) section 1.2 and will land in follow-up files
under this same namespace.

### Token keys

A token carries up to seven optional keys, each granting authority over a specific operation. None
of these can be *added* via `TokenUpdate` if they were not set at creation time — leaving a key
unset at create permanently disables that capability (HAPI: `KEY_NOT_PROVIDED`). The keys can be
*rotated* by an update if they were set originally, and (under HIP-540) cleared to an immutable
state.

| Key            | Grants the authority to                                                                          |
|----------------|--------------------------------------------------------------------------------------------------|
| `adminKey`     | update token metadata (`TokenUpdate`) and delete the token (`TokenDelete`). Unset → immutable.   |
| `supplyKey`    | mint and burn units (`TokenMint`, `TokenBurn`). Unset → fixed supply after creation.             |
| `kycKey`       | grant / revoke KYC (`TokenGrantKyc`, `TokenRevokeKyc`). Unset → KYC concept disabled.            |
| `freezeKey`    | freeze / unfreeze accounts for this token (`TokenFreeze`, `TokenUnfreeze`). Unset → no freeze.   |
| `wipeKey`      | wipe units from a non-treasury account (`TokenWipe`). Unset → wipe disabled.                     |
| `pauseKey`     | pause / unpause every operation on the token (`TokenPause`, `TokenUnpause`). Unset → unpausable. |
| `feeScheduleKey` | update the token's custom fee schedule (`TokenFeeScheduleUpdate`). Unset → fees immutable.     |
| `metadataKey`  | update NFT-level metadata after mint (`TokenUpdateNfts`, HIP-657). Unset → metadata immutable.   |

The `metadata` field on the token itself (and on individual NFT serials) is opaque bytes — typical
uses are an IPFS CID, an HTTPS URL, or an inlined JSON manifest. The token's `decimals`,
`initialSupply`, and `maxSupply` fields are expressed in the smallest indivisible unit (e.g. for a
6-decimals USDC clone, `1_000_000` means "one display unit").

### Treasury account

Every token has a `treasuryAccountId`: the account that receives `initialSupply` at create time and
that all subsequent `TokenMint` units are credited to. Burns also draw from the treasury. The
treasury account must have been associated with the token (implicitly true at create — the protocol
auto-associates the treasury). Treasury rotation via `TokenUpdate` requires both the *current* and
the *new* treasury to sign.

### Token associations

Before a non-treasury account can hold a positive balance of a token it must explicitly *associate*
with it (`TokenAssociate`). Accounts created with `maxAutomaticTokenAssociations > 0` may instead
auto-associate on first inbound transfer up to that limit (HIP-23 / HIP-904); `TokenAssociate`
remains the explicit path. `TokenDissociate` removes an association — the protocol requires the
account to hold zero balance of the token (and zero NFTs of that collection) at dissociate time,
otherwise the transaction fails.

A token's lifecycle is:

```
                                          ┌── TokenMint    ──┐
  create ── (associate | dissociate)* ── ─┤                  ├── update* ── delete
                                          └── TokenBurn    ──┘
```

### Signing requirements (token lifecycle)

| Transaction       | Signers required                                                                                                                                                                                                                                                                                                |
|-------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `TokenCreate`     | the *payer*; **and** the `treasuryAccountId` (auto-association of the treasury implicitly binds the account); **and** the `adminKey` if it is set (anti-spoofing: cannot bind another party's key as admin without their consent); **and** the `autoRenewAccount` if it is set and differs from the payer / treasury. Other keys (`supplyKey`, `kycKey`, ...) do **not** need to sign creation — they gate only future permissioned operations. |
| `TokenUpdate`     | the *payer*; **and** the token's *current* `adminKey`; **and**, if a key field is being rotated to a new value, the *new* key (anti-spoofing); **and**, if `treasuryAccountId` is being changed, both the *current* and the *new* treasury accounts; **and**, if `autoRenewAccount` is being changed, the new account. |
| `TokenDelete`     | the *payer*; **and** the token's current `adminKey` (an immutable token — `adminKey` unset — cannot be deleted; the request fails with `TOKEN_IS_IMMUTABLE`).                                                                                                                                                   |
| `TokenAssociate`  | the *payer*; **and** the `accountId` being associated (the account opting in to hold the token). The token's keys are **not** consulted — anyone may associate to any token.                                                                                                                                    |
| `TokenDissociate` | the *payer*; **and** the `accountId` being dissociated.                                                                                                                                                                                                                                                         |
| `TokenMint`       | the *payer*; **and** the token's `supplyKey` (mint is impossible if `supplyKey` is unset — `TOKEN_HAS_NO_SUPPLY_KEY`).                                                                                                                                                                                          |
| `TokenBurn`       | the *payer*; **and** the token's `supplyKey`. Burns always draw from the treasury account; the treasury itself does not need to co-sign.                                                                                                                                                                        |

When the operator holds every required key (e.g. operator = treasury = admin) `signWithOperatorAndSubmit(client)`
is sufficient. Multi-key flows go through the `signWithOperator(client).sign(...)` pattern shown
in [`transactions-accounts.md`](transactions-accounts.md).

The general rule (as in topics): a key field on a create transaction must sign **if** binding it
would grant that key authority over a real asset (`adminKey`, `treasuryAccountId`,
`autoRenewAccount`). Keys that only gate *future* permissioned operations and confer no liability
right now (`supplyKey`, `kycKey`, `freezeKey`, `wipeKey`, `pauseKey`, `feeScheduleKey`,
`metadataKey`) do not need to sign creation.

### Mutability and "clearing" keys / auto-renew

`TokenUpdate` distinguishes "leave unchanged" from "clear":

- A `@@nullable` field set to **null** (or never set on the builder) means *leave unchanged*.
- Clearing an optional `Address`-typed field (`autoRenewAccount`, `treasuryAccountId` is
  non-clearable) uses [`ZERO_ADDRESS`](../base/ledger.md) as the sentinel value.
- Clearing an optional `Key`-typed field (any of the seven token keys) currently has no portable
  representation in this V3 spec because the `Key` / `KeyList` sum type is missing — see
  *Questions & Comments*. HIP-540 added the ability to clear *any* lower-privilege key to an
  irrevocably-empty state; full support depends on the `Key` sum type landing.

## API Schema

```
namespace consensusnode.transactions.tokens
requires {Address, AccountId} from ledger
requires {PublicKey} from keys
requires {TokenType, TokenSupplyType} from token
requires {Receipt, Transaction} from consensusnode.transactions

// Creates a new token. The token is identified by the `tokenId` returned in the receipt. All
// seven key fields are optional and follow the rule that a key not set at create time can never
// be added later (HAPI: `KEY_NOT_PROVIDED`); leaving e.g. `supplyKey` unset permanently fixes
// the token's supply at `initialSupply`.
@@finalType
TokenCreateTransaction extends Transaction<TokenCreateReceipt> {
    @@immutable @@maxLength(100) name: string                       // human-readable token name
    @@immutable @@maxLength(100) symbol: string                     // short ticker / symbol
    @@immutable tokenType: TokenType                                // FUNGIBLE_COMMON | NON_FUNGIBLE_UNIQUE; cannot be changed later
    @@immutable @@default(0) decimals: int32                        // FUNGIBLE_COMMON only: precision of the smallest indivisible unit (e.g. 6 for USDC); MUST be 0 for NON_FUNGIBLE_UNIQUE
    @@immutable @@default(0) initialSupply: int64                   // FUNGIBLE_COMMON only: amount minted to the treasury at create (in smallest unit); MUST be 0 for NON_FUNGIBLE_UNIQUE
    @@immutable treasuryAccountId: AccountId                        // account receiving `initialSupply` and all subsequent mints; MUST sign the create transaction
    @@immutable @@default(INFINITE) supplyType: TokenSupplyType     // INFINITE | FINITE; cannot be changed later
    @@immutable @@nullable @@min(1) maxSupply: int64                // FINITE only: protocol-enforced ceiling on totalSupply; unset (and only valid) when supplyType is INFINITE
    @@immutable @@default(false) freezeDefault: bool                // when true, newly associated accounts start in the FROZEN state (require an explicit TokenUnfreeze before they can move balances); requires freezeKey to be set
    @@immutable @@nullable @@maxLength(100) tokenMemo: string       // short, human-readable label
    @@immutable @@default([]) metadata: bytes                       // opaque token-level metadata (e.g. IPFS CID, HTTPS URL, JSON manifest)
    @@immutable @@nullable expirationTime: zonedDateTime            // when the token expires; SDK default applies when unset
    @@immutable @@nullable autoRenewPeriod: seconds                 // renewal window applied when expiration approaches; protocol default applies when unset
    @@immutable @@nullable autoRenewAccount: AccountId              // account paying for auto-renewal; MUST sign the create transaction if set and not equal to the treasury / payer

    // Optional key fields — see "Token keys" in the description above for what each one grants.
    // A key NOT set at create time CANNOT be added by TokenUpdate.
    @@immutable @@nullable adminKey: PublicKey                      // controls update / delete; unset → immutable token; MUST sign the create transaction when set (anti-spoofing)
    @@immutable @@nullable supplyKey: PublicKey                     // controls mint / burn; unset → fixed supply after creation
    @@immutable @@nullable kycKey: PublicKey                        // controls KYC grant / revoke; unset → KYC disabled
    @@immutable @@nullable freezeKey: PublicKey                     // controls freeze / unfreeze; unset → no freeze
    @@immutable @@nullable wipeKey: PublicKey                       // controls wipe of non-treasury balances; unset → wipe disabled
    @@immutable @@nullable pauseKey: PublicKey                      // controls pause / unpause; unset → unpausable
    @@immutable @@nullable feeScheduleKey: PublicKey                // controls custom-fee schedule updates; unset → fees immutable
    @@immutable @@nullable metadataKey: PublicKey                   // controls per-serial NFT metadata updates (HIP-657); unset → metadata immutable
}

@@finalType
TokenCreateReceipt extends Receipt {
    @@immutable tokenId: Address                                    // id of the newly created token
}

// Updates one or more of a token's mutable fields. Every nullable field is "leave unchanged"
// when null. `tokenType`, `supplyType`, `decimals`, and `initialSupply` are NOT updatable — they
// are fixed at create.
//
// Rotating a key field to a new key requires the new key to also sign (anti-spoofing). A key
// field that was unset at create cannot be added here (HAPI: `KEY_NOT_PROVIDED`). Clearing a
// key field to make the corresponding capability permanently disabled is allowed per HIP-540 but
// has no portable representation until the `Key` sum type lands — see Questions & Comments.
@@finalType
TokenUpdateTransaction extends Transaction<TokenUpdateReceipt> {
    @@immutable tokenId: Address                                    // the token being updated
    @@immutable @@nullable @@maxLength(100) name: string
    @@immutable @@nullable @@maxLength(100) symbol: string
    @@immutable @@nullable treasuryAccountId: AccountId             // when set, replaces the treasury; both the OLD and the NEW treasury MUST sign
    @@immutable @@nullable @@maxLength(100) tokenMemo: string
    @@immutable @@nullable metadata: bytes                          // when set, replaces the token-level metadata
    @@immutable @@nullable expirationTime: zonedDateTime
    @@immutable @@nullable autoRenewPeriod: seconds
    @@immutable @@nullable autoRenewAccount: AccountId              // when set, replaces the auto-renew account; the new account MUST also sign

    // Key rotations — each requires the new key to also sign (anti-spoofing). A key that was
    // unset at create cannot be set here.
    @@immutable @@nullable adminKey: PublicKey
    @@immutable @@nullable supplyKey: PublicKey
    @@immutable @@nullable kycKey: PublicKey
    @@immutable @@nullable freezeKey: PublicKey
    @@immutable @@nullable wipeKey: PublicKey
    @@immutable @@nullable pauseKey: PublicKey
    @@immutable @@nullable feeScheduleKey: PublicKey
    @@immutable @@nullable metadataKey: PublicKey
}

@@finalType
TokenUpdateReceipt extends Receipt {
}

// Deletes a token. Only possible if the token has an adminKey (an immutable token cannot be
// deleted). After deletion, the token id is not reusable; transactions referencing it return
// `TOKEN_WAS_DELETED`. Existing balances are NOT swept by this transaction — they remain
// recorded but unusable for transfer.
@@finalType
TokenDeleteTransaction extends Transaction<TokenDeleteReceipt> {
    @@immutable tokenId: Address
}

@@finalType
TokenDeleteReceipt extends Receipt {
}

// Associates an account with one or more tokens so that the account may receive non-zero
// balances of them. The account being associated MUST sign the transaction (it is opting in;
// the token's keys are not consulted). Accounts configured with
// `maxAutomaticTokenAssociations > 0` may auto-associate on first inbound transfer instead;
// `TokenAssociate` remains the explicit path for accounts that do not, or that exceeded their
// auto-association budget.
@@finalType
TokenAssociateTransaction extends Transaction<TokenAssociateReceipt> {
    @@immutable accountId: AccountId                                // the account opting in to hold the listed tokens (MUST sign)
    @@immutable @@maxLength(100) tokens: set<Address>               // tokens to associate; duplicates are protocol-rejected (TOKEN_ID_REPEATED_IN_TOKEN_LIST) and order is irrelevant — hence a set, not a list
}

@@finalType
TokenAssociateReceipt extends Receipt {
}

// Removes the association between an account and one or more tokens. The protocol requires the
// account to hold zero balance of every listed token (and zero NFTs of any listed NFT
// collection) at consensus time, otherwise the transaction fails with
// `TRANSACTION_REQUIRES_ZERO_TOKEN_BALANCES`. Use `TransferTransaction` first to drain the
// account if necessary.
@@finalType
TokenDissociateTransaction extends Transaction<TokenDissociateReceipt> {
    @@immutable accountId: AccountId                                // the account opting out (MUST sign)
    @@immutable @@maxLength(100) tokens: set<Address>               // tokens to dissociate; same set semantics as TokenAssociateTransaction.tokens
}

@@finalType
TokenDissociateReceipt extends Receipt {
}

// Mints new units of a token, crediting them to the treasury account. Requires the token's
// supplyKey to sign. The payload differs by `tokenType`:
//
//   - FUNGIBLE_COMMON  → set `amount` (count in the smallest indivisible unit, must be > 0).
//   - NON_FUNGIBLE_UNIQUE → set `metadata` (one entry per new serial, max 10 per transaction);
//                           the consensus node assigns sequential serial numbers and returns
//                           them in the receipt.
//
// Mixing the two is rejected by the network; the @@oneOf at the type level makes this
// statically checkable.
@@oneOf(amount, metadata)
@@finalType
TokenMintTransaction extends Transaction<TokenMintReceipt> {
    @@immutable tokenId: Address
    @@immutable @@nullable @@min(1) amount: int64                   // FUNGIBLE_COMMON only: amount to mint (in smallest unit)
    @@immutable @@nullable @@maxLength(10) metadata: list<bytes>    // NON_FUNGIBLE_UNIQUE only: one metadata blob per new serial; the protocol caps the list at 10 per transaction
}

@@finalType
TokenMintReceipt extends Receipt {
    @@immutable totalSupply: int64                                  // the token's new totalSupply after this mint
    @@immutable @@default([]) serials: list<int64>                  // NON_FUNGIBLE_UNIQUE only: the serial numbers assigned to the newly minted NFTs (empty for FUNGIBLE_COMMON)
}

// Burns existing units of a token from the treasury account. Requires the token's supplyKey to
// sign. The payload differs by `tokenType`:
//
//   - FUNGIBLE_COMMON  → set `amount` (must be > 0 and ≤ treasury balance).
//   - NON_FUNGIBLE_UNIQUE → set `serials` (the serials to burn, each currently held by the
//                           treasury account; max 10 per transaction).
//
// Burning NFT serials held by a non-treasury account is not possible via this transaction —
// use `TokenWipe` (gated by `wipeKey`) instead. Mixing fungible and NFT payloads is rejected;
// the @@oneOf at the type level makes this statically checkable.
@@oneOf(amount, serials)
@@finalType
TokenBurnTransaction extends Transaction<TokenBurnReceipt> {
    @@immutable tokenId: Address
    @@immutable @@nullable @@min(1) amount: int64                   // FUNGIBLE_COMMON only: amount to burn (in smallest unit)
    @@immutable @@nullable @@maxLength(10) serials: list<int64>     // NON_FUNGIBLE_UNIQUE only: serials to burn (must be held by the treasury); the protocol caps the list at 10 per transaction
}

@@finalType
TokenBurnReceipt extends Receipt {
    @@immutable totalSupply: int64                                  // the token's new totalSupply after this burn
}
```

## Examples

### Create a fungible token (operator is admin, treasury, and supply key)

The simplest case: a 6-decimals stablecoin-style token with a fixed-but-mintable supply. The
operator pays, is the treasury, and holds the admin + supply keys, so a single
`signWithOperatorAndSubmit` covers every required signature.

```
HieroClient client = ...;

Response<TokenCreateReceipt> response = new TokenCreateTransaction()
    .name("Example USD")
    .symbol("EUSD")
    .tokenType(TokenType.FUNGIBLE_COMMON)
    .decimals(6)
    .initialSupply(1_000_000_000_000L)         // 1,000,000.000000 EUSD to the treasury
    .treasuryAccountId(client.operatorAccountId)
    .supplyType(TokenSupplyType.INFINITE)
    .adminKey(client.operatorPublicKey)
    .supplyKey(client.operatorPublicKey)
    .signWithOperatorAndSubmit(client);

Address tokenId = response.queryReceipt().tokenId;
```

### Create a finite-supply NFT collection

`tokenType = NON_FUNGIBLE_UNIQUE` forces `decimals = 0` and `initialSupply = 0`. Serials are
allocated by `TokenMint` calls, capped at `maxSupply` once `supplyType = FINITE`.

```
new TokenCreateTransaction()
    .name("Example Art")
    .symbol("EART")
    .tokenType(TokenType.NON_FUNGIBLE_UNIQUE)
    .treasuryAccountId(client.operatorAccountId)
    .supplyType(TokenSupplyType.FINITE)
    .maxSupply(10_000L)
    .adminKey(client.operatorPublicKey)
    .supplyKey(client.operatorPublicKey)
    .metadataKey(client.operatorPublicKey)     // allow per-serial metadata updates (HIP-657)
    .signWithOperatorAndSubmit(client);
```

### Treasury is a separate account (multi-signature)

The operator pays the fee, but a dedicated treasury account custodies the supply. The treasury
must co-sign — its signature is what binds the treasury to the new token.

```
PackedTransaction<...> packed = new TokenCreateTransaction()
    .name("Acme Loyalty")
    .symbol("ACME")
    .tokenType(TokenType.FUNGIBLE_COMMON)
    .decimals(0)
    .initialSupply(0)
    .treasuryAccountId(treasury.accountId)
    .supplyKey(supplyOwnerPublicKey)
    .adminKey(client.operatorPublicKey)
    .signWithOperator(client)                  // payer + adminKey (operator)
    .sign(treasury);                           // treasury authorises being bound to the token

packed.submit(client);
```

### Associate before receiving the token

Bob must associate with `tokenId` before any `TransferTransaction` can credit him. He pays his
own fee here, so a single `sign(...)` is enough.

```
new TokenAssociateTransaction()
    .accountId(bob.accountId)
    .tokens([tokenId])
    .sign(bob, nodes)
    .submit(client);
```

### Mint additional fungible supply

The treasury automatically receives the new units. The `supplyKey` (here the operator) must
sign.

```
Response<TokenMintReceipt> response = new TokenMintTransaction()
    .tokenId(tokenId)
    .amount(500_000_000L)                      // 500.000000 additional EUSD
    .signWithOperatorAndSubmit(client);

int64 newTotalSupply = response.queryReceipt().totalSupply;
```

### Mint NFT serials in a single batch

Up to 10 metadata blobs may be supplied per transaction; the protocol assigns sequential serial
numbers and returns them in the receipt.

```
Response<TokenMintReceipt> response = new TokenMintTransaction()
    .tokenId(nftCollectionId)
    .metadata([
        "ipfs://QmHash1".bytes,
        "ipfs://QmHash2".bytes,
        "ipfs://QmHash3".bytes,
    ])
    .signWithOperatorAndSubmit(client);

list<int64> newSerials = response.queryReceipt().serials;   // e.g. [1, 2, 3]
```

### Burn fungible / NFT supply

Burning draws from the treasury account; the treasury itself does not need to co-sign — the
supply key authorises the burn.

```
// Fungible:
new TokenBurnTransaction()
    .tokenId(tokenId)
    .amount(10_000_000L)
    .signWithOperatorAndSubmit(client);

// NFT (serials must be currently held by the treasury):
new TokenBurnTransaction()
    .tokenId(nftCollectionId)
    .serials([1L, 2L])
    .signWithOperatorAndSubmit(client);
```

### Rotate the supply key (multi-signature)

Both the *current* and the *new* supply key must sign, in addition to the payer and admin key.

```
PackedTransaction<...> packed = new TokenUpdateTransaction()
    .tokenId(tokenId)
    .supplyKey(newSupplyPublicKey)
    .signWithOperator(client)                  // payer + adminKey (operator) + current supplyKey (operator)
    .sign(newSupplySigner);                    // new supplyKey

packed.submit(client);
```

### Dissociate and delete

Bob must drain his balance first; then he can dissociate. Once the admin decides the token has
reached the end of its life, the admin key deletes it.

```
// 1. Drain Bob (omitted: TransferTransaction back to treasury).
// 2. Dissociate:
new TokenDissociateTransaction()
    .accountId(bob.accountId)
    .tokens([tokenId])
    .sign(bob, nodes)
    .submit(client);

// 3. Delete the token (admin key signs):
new TokenDeleteTransaction()
    .tokenId(tokenId)
    .signWithOperatorAndSubmit(client);
```

## Questions & Comments

- **All key fields are typed as `PublicKey`, not `Key`.** Same placeholder typing as in
  [`transactions-files.md`](transactions-files.md) and
  [`transactions-topics.md`](transactions-topics.md): HAPI takes a `Key` (sum type over
  `PublicKey` / `ContractID` / `DelegatableContractID` / `KeyList` / `ThresholdKey`), so the
  current single-key shape cannot express m-of-n custody, contract-controlled tokens (where a
  smart contract is the supply / KYC / freeze authority), or nested key structures. The `Key`
  sum type is tracked in [`missing-features.md`](../../missing-features.md) section 3.2 as the
  prerequisite; once it lands, every key field in this file should be retyped to `Key`. This is
  particularly load-bearing for tokens: real-world deployments routinely guard `supplyKey` with
  a treasury-DAO threshold key or a contract id, not a single public key.

- **Clearing a key field on `TokenUpdate` — not yet expressible.** HIP-540 lets an admin (or
  the key holder itself) clear any of the seven lower-privilege key fields to an
  irrevocably-empty state (e.g. drop the `adminKey` to make a token immutable, drop the
  `wipeKey` to permanently disable wipe). HAPI signals "clear" via an empty `KeyList` sentinel;
  V3 cannot express this until `Key` / `KeyList` exist (see
  [`missing-features.md`](../../missing-features.md) section 3.2). For
  `Address`-typed fields, [`ZERO_ADDRESS`](../base/ledger.md) is the existing sentinel —
  `autoRenewAccount(ZERO_ADDRESS)` drops the auto-renew account. `treasuryAccountId` is **not**
  clearable: a token must always have a treasury.

- **`customFees` / `feeScheduleKey` interaction is partial.** This file exposes `feeScheduleKey`
  on `TokenCreate` / `TokenUpdate` because the field is independent and cheap to model, but the
  write-side `CustomFee` hierarchy needed by `TokenCreate.customFees` and by
  `TokenFeeScheduleUpdate` is **not** specified here. Read-side custom-fee shapes live in
  [`mirror-node-token.md`](../mirror-node-client/mirror-node-token.md); the write-side builder is
  tracked in [`missing-features.md`](../../missing-features.md) section 3.3 and depends on the
  same `Key` sum type. Adding `customFees: list<CustomFee>` here once the write-side model lands
  is additive and breaks nothing.

- **`TokenWipe`, `Freeze` / `Unfreeze`, `Pause` / `Unpause`, `GrantKyc` / `RevokeKyc`,
  `FeeScheduleUpdate`, `UpdateNfts` are deliberately out of scope of this file.** They live in
  the same namespace and follow the same lifecycle / signing model as the seven transactions
  here — splitting the file keeps the core lifecycle readable. Each gets its own file as it
  lands. Tracked in [`missing-features.md`](../../missing-features.md) section 1.2.

- **HIP-904 airdrop transactions are deliberately out of scope of this file.**
  `TokenAirdrop` / `TokenClaimAirdrop` / `TokenCancelAirdrop` / `TokenReject` reuse the
  association rules described here but interact with `PendingAirdropId` and the new
  `pendingAirdrops` mirror-node domain; they deserve their own spec rather than being smuggled
  into the lifecycle file. Tracked in [`missing-features.md`](../../missing-features.md) section
  1.2.

- **`TokenMint` / `TokenBurn` cap at 10 NFTs per transaction.** That is the current HAPI limit
  (`TOKEN_MAX_BATCH_SIZE_REACHED`) and is encoded as `@@maxLength(10)` on the `metadata` and
  `serials` lists. If a future HIP raises the cap (HIP-1300 jumbo transactions touch this area),
  the annotation widens — callers that respected the previous cap remain valid.