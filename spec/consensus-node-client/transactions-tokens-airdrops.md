# Token Airdrop Transactions API (HIP-904)

This file extends the HTS token service ([`transactions-tokens.md`](transactions-tokens.md)) with
the **HIP-904 frictionless-airdrop** transactions. They share the
`consensusnode.transactions.tokens` namespace and the `Transaction<$$Receipt>` lifecycle.

## Description

A normal token transfer fails if the recipient has neither associated the token nor a free
auto-association slot. HIP-904 adds a *push* model that never fails for that reason:

- **`TokenAirdrop`** — transfers tokens to recipients exactly like a `TransferTransaction`'s token
  legs, **but** if a recipient is not associated and has no free auto-association slot, the
  transfer is parked as a **pending airdrop** instead of failing. Associated recipients (or those
  with a free slot) receive immediately. The set of pending airdrops created lands on the
  transaction **record** (not the receipt) — see *Questions & Comments*.
- **`TokenClaimAirdrop`** — the **recipient** accepts one or more pending airdrops, auto-associating
  and crediting the tokens. Signed by the receiver.
- **`TokenCancelAirdrop`** — the **sender** withdraws one or more of its own outstanding pending
  airdrops before they are claimed. Signed by the sender.
- **`TokenReject`** — the current **holder** returns unwanted tokens to the token's treasury at no
  cost (the anti-spam counterpart: reject already-held tokens). Signed by the holder.

Airdrop transfers reuse the same [`TokenTransfer`](transactions-accounts.md) /
[`NftTransfer`](transactions-accounts.md) leg types as `TransferTransaction` (HAPI reuses
`TokenTransferList`); HBAR is not airdroppable, so there is no `hbarTransfers` leg.

### Pending-airdrop identity

A pending airdrop is identified by `(senderId, receiverId, token)` — fungible by token id, or NFT
by token id + serial. HAPI types this as `PendingAirdropId`; V3 does not yet have that typed
identifier (nor `NftId`), so it is modelled here as the `PendingAirdrop` shape below with a
`@@nullable serial`. Both are tracked in [`missing-features.md`](../../missing-features.md) §3.1.

### Signing requirements

| Transaction | Signers required |
|-------------|------------------|
| `TokenAirdrop` | the *payer*; **and** each debited sender (or the payer, per leg, when `approved = true` draws on an allowance) — identical to `TransferTransaction`'s token / NFT legs. |
| `TokenClaimAirdrop` | the *payer*; **and** each pending airdrop's `receiverId` (the account accepting delivery). |
| `TokenCancelAirdrop` | the *payer*; **and** each pending airdrop's `senderId` (the account withdrawing its offer). |
| `TokenReject` | the *payer*; **and** the `owner` returning the tokens (defaults to the payer when unset). |

## API Schema

```
namespace consensusnode.transactions.tokens
requires {Address, AccountId} from ledger
requires {Receipt, Transaction} from consensusnode.transactions
requires {TokenTransfer, NftTransfer} from consensusnode.transactions.accounts

// A single pending airdrop, identified by its sender, receiver, and token. `serial` is null for a
// fungible airdrop and set for a specific NFT. Placeholder for HAPI's typed PendingAirdropId
// (see missing-features §3.1).
type PendingAirdrop {
    @@immutable senderId: AccountId
    @@immutable receiverId: AccountId
    @@immutable tokenId: Address
    @@immutable @@nullable serial: int64        // null → fungible; set → that NFT serial
}

// A reference to tokens being rejected: either a whole fungible token (serial null) or a single
// NFT serial. Placeholder for HAPI's TokenReference / NftId (see missing-features §3.1).
type TokenReference {
    @@immutable tokenId: Address
    @@immutable @@nullable serial: int64        // null → fungible token; set → that NFT serial
}

// Transfers tokens to recipients; recipients that are not associated and have no free
// auto-association slot receive a pending airdrop instead of the transaction failing. Reuses the
// transfer leg types from TransferTransaction. The protocol caps the combined transfer list at 10.
// The pending airdrops created are reported on the record, not this receipt (see Q&C).
@@finalType
TokenAirdropTransaction extends Transaction<TokenAirdropReceipt> {
    @@immutable @@default([]) tokenTransfers: list<TokenTransfer>   // fungible legs (sum per token must be zero)
    @@immutable @@default([]) nftTransfers: list<NftTransfer>       // NFT legs
}

@@finalType
TokenAirdropReceipt extends Receipt {
}

// The receiver accepts pending airdrops, auto-associating and crediting the tokens. Each receiver
// must sign. Protocol cap 10 per transaction; no duplicates.
@@finalType
TokenClaimAirdropTransaction extends Transaction<TokenClaimAirdropReceipt> {
    @@immutable @@minLength(1) @@maxLength(10) pendingAirdrops: list<PendingAirdrop>
}

@@finalType
TokenClaimAirdropReceipt extends Receipt {
}

// The sender withdraws its own outstanding pending airdrops before they are claimed. Each sender
// must sign. Protocol cap 10 per transaction; no duplicates.
@@finalType
TokenCancelAirdropTransaction extends Transaction<TokenCancelAirdropReceipt> {
    @@immutable @@minLength(1) @@maxLength(10) pendingAirdrops: list<PendingAirdrop>
}

@@finalType
TokenCancelAirdropReceipt extends Receipt {
}

// Returns unwanted, already-held tokens to their treasury at no cost to the holder. Protocol cap
// 10 references per transaction.
@@finalType
TokenRejectTransaction extends Transaction<TokenRejectReceipt> {
    @@immutable @@nullable owner: AccountId                          // the holder returning the tokens; defaults to the payer when unset
    @@immutable @@minLength(1) @@maxLength(10) rejections: list<TokenReference>
}

@@finalType
TokenRejectReceipt extends Receipt {
}
```

## Examples

### Airdrop a fungible token (parks as pending if the recipient is not associated)

```
HieroClient client = ...;

new TokenAirdropTransaction()
    .tokenTransfers([
        new TokenTransfer(usdcTokenId, treasury.accountId, -100, 6, false),
        new TokenTransfer(usdcTokenId, recipient,          +100, 6, false),
    ])
    .signWithOperatorAndSubmit(client);
```

### Recipient claims a pending airdrop

```
new TokenClaimAirdropTransaction()
    .pendingAirdrops([
        new PendingAirdrop(treasury.accountId, me.accountId, usdcTokenId, /* serial */ null),
    ])
    .sign(me, nodes)
    .submit(client);
```

### Reject already-held tokens back to treasury

```
new TokenRejectTransaction()
    .owner(me.accountId)
    .rejections([
        new TokenReference(usdcTokenId, null),          // a fungible token
        new TokenReference(nftCollectionId, 42L),       // one NFT serial
    ])
    .sign(me, nodes)
    .submit(client);
```

## Questions & Comments

- **Pending airdrops are a *record* field, not a receipt field.** HAPI returns the set of created
  pending airdrops in `TransactionRecord.new_pending_airdrops`, not the receipt. V3 does not yet
  type the record per transaction, so `TokenAirdropReceipt` is empty and the pending-airdrop result
  is part of the open base-`Record` design decision tracked in
  [`missing-features.md`](../../missing-features.md) §3.5. Add a `pendingAirdrops` field to a typed
  `TokenAirdropRecord` once that lands.

- **`PendingAirdrop` / `TokenReference` are placeholders for typed identifiers.** They stand in for
  HAPI's `PendingAirdropId` and `TokenReference` / `NftId`, which are not yet typed in V3 (§3.1).
  The `@@nullable serial` discriminates fungible vs NFT until `NftId` lands, at which point both
  shapes should be retyped.

- **Reusing `TokenTransfer` / `NftTransfer` from `consensusnode.transactions.accounts`.** HAPI
  shares `TokenTransferList` between `CryptoTransfer` and `TokenAirdrop`; V3 currently defines those
  leg types in [`transactions-accounts.md`](transactions-accounts.md), so this file imports them
  from there. Arguably those leg types belong in a shared location (they are not crypto-service
  specific); if they move, this import follows. The dependency is one-directional
  (`...tokens → ...accounts`) and acyclic.

- **`TokenReject` `owner` defaults to the payer.** Matching HAPI, an unset `owner` means the payer
  is rejecting its own tokens; set it only when a separate account (which must then sign) is the
  holder.
