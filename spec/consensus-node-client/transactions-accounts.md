# Account Transactions API


## Description

This namespace groups all crypto-service transactions that operate on accounts: lifecycle
(`AccountCreate` / `AccountUpdate` / `AccountDelete`), value movement
(`TransferTransaction` — HBAR, fungible tokens, and NFTs in one atomic transaction), and
delegated-spending authorization (`AccountAllowanceApproveTransaction` /
`AccountAllowanceDeleteTransaction`).

`TransferTransaction` is the single transaction type for *all* value transfers on the network.
A single instance can mix HBAR, fungible-token, and NFT movements; the consensus node settles
all movements atomically (either every leg succeeds or none does). Each leg can be marked
`approved = true` to indicate the debit should be drawn from an existing allowance granted to
the payer rather than directly from the named account; in that case the payer must hold a
sufficient allowance from the owner, and the owner does not need to sign.

#### Signing requirements (transfer)

Every leg that *removes* value from an account requires that account's signature, unless the
leg is `approved = true` (in which case the *spender* — the transaction payer — signs in the
owner's place using a previously granted allowance):

| Leg                                  | Signer required                              |
|--------------------------------------|----------------------------------------------|
| `HbarTransfer` debit (`amount < 0`)  | the debited `accountId`, or payer if `approved` |
| `HbarTransfer` credit (`amount > 0`) | none — unless the recipient has `receiverSignatureRequired = true` |
| `TokenTransfer` debit                | the debited `accountId`, or payer if `approved` |
| `TokenTransfer` credit               | none — unless `receiverSignatureRequired = true` |
| `NftTransfer`                        | `sender`, or payer if `approved`             |
| Transaction fee                      | the *payer* (whichever account is bound by `pack` / `sign…`) |

In practice this means a transfer where the operator moves value out of someone else's account
is a **multi-signature** flow: the operator signs as payer, the owner signs to authorize the
debit. The single-call `signWithOperator…` methods are only sufficient when the operator is
the only account being debited (or every debit leg is `approved`).

`AccountAllowanceApproveTransaction` grants delegated-spend rights from an *owner* account to
a *spender* account. Three independent flavours exist:

- **HBAR allowances** — the spender may transfer up to `amount` HBAR from the owner.
- **Fungible-token allowances** — the spender may transfer up to `amount` units of `tokenId`
  from the owner.
- **NFT allowances** — either a list of specific `serials` of `tokenId` (per-serial grant)
  *or* `approveAll = true` (a blanket grant covering every current and future serial of
  `tokenId` the owner holds). The `delegatingSpender` field supports HIP-336 sub-approvals:
  a spender that already holds an `approveAll` grant may delegate per-serial rights onward.

Re-running approve with the same `(owner, spender, asset)` triple overwrites the prior amount
(for HBAR / fungible tokens) — an amount of `0` revokes the allowance. NFT per-serial grants
accumulate; `approveAll = false` revokes the blanket grant for that `(owner, spender, tokenId)`.

`AccountAllowanceDeleteTransaction` is **NFT-only**: it lets the *owner* revoke previously
granted per-serial NFT allowances independently of any specific spender. HBAR and
fungible-token allowances are not deletable through this transaction; revoke them by issuing
an approve with `amount = 0`. Blanket `approveAll = true` grants are revoked by issuing an
approve with `approveAll = false` for the same `(owner, spender, tokenId)`.

## API Schema

```
namespace consensusnode.transactions.accounts
requires {Address, AccountId} from ledger
requires {PublicKey} from keys
requires {NativeToken} from nativeToken
requires {Receipt, Transaction} from consensusnode.transactions

@@finalType
AccountCreateTransaction extends Transaction<AccountCreateReceipt> {
    @@immutable key: PublicKey
    @@immutable @@default(0) initialBalance: NativeToken<ANY, ANY>
    @@immutable @@nullable accountMemo: string
    @@immutable @@default(false) receiverSignatureRequired: bool
    @@immutable @@nullable maxAutomaticTokenAssociations: int32
    @@immutable @@nullable autoRenewPeriod: seconds
    @@immutable @@nullable stakedAccountId: AccountId
    @@immutable @@nullable stakedNodeId: int64
    @@immutable @@default(false) declineStakingReward: bool
    @@immutable @@nullable alias: bytes
}

@@finalType
AccountCreateReceipt extends Receipt {
    @@immutable accountId: AccountId
}

@@finalType
AccountUpdateTransaction extends Transaction<AccountUpdateReceipt> {
    @@immutable accountId: AccountId                            // the account that is being updated
    @@immutable @@nullable key: PublicKey                          // the new key (requires signatures with both old and new keys)
    @@immutable @@nullable accountMemo: string
    @@immutable @@nullable receiverSignatureRequired: bool
    @@immutable @@nullable maxAutomaticTokenAssociations: int32
    @@immutable @@nullable autoRenewPeriod: seconds
    @@immutable @@nullable expirationTime: zonedDateTime
    @@immutable @@nullable stakedAccountId: AccountId            // mutually exclusive with stakedNodeId
    @@immutable @@nullable stakedNodeId: int64                          // mutually exclusive with stakedAccountId
    @@immutable @@nullable declineStakingReward: bool
}

@@finalType
AccountUpdateReceipt extends Receipt {
}

@@finalType
AccountDeleteTransaction extends Transaction<AccountDeleteReceipt> {
    @@immutable accountId: AccountId                            // the account that is being deleted (must sign)
    @@immutable transferAccountId: AccountId                    // the account that receives the remaining hbar balance
}

@@finalType
AccountDeleteReceipt extends Receipt {
}

// One leg of an HBAR transfer. The consensus node requires the sum of all `amount` values in
// `TransferTransaction.hbarTransfers` to equal zero (debits balance credits exactly).
type HbarTransfer {
    @@immutable accountId: AccountId                // the account being debited or credited
    @@immutable amount: NativeToken<ANY, ANY>       // signed amount: negative = debit, positive = credit
    @@immutable @@default(false) approved: bool     // when true, the debit is drawn from an allowance granted to the transaction payer; the owner (accountId) need not sign
}

// One leg of a fungible-token transfer. The sum of `amount` per `tokenId` across all
// `TransferTransaction.tokenTransfers` must equal zero.
type TokenTransfer {
    @@immutable tokenId: Address                    // the fungible token being moved
    @@immutable accountId: AccountId                // the account being debited or credited
    @@immutable amount: int64                       // signed amount in the token's smallest unit: negative = debit, positive = credit
    @@immutable @@nullable expectedDecimals: int32  // optional protocol-side decimals check; if set and the token's actual decimals differ, the network rejects the transaction
    @@immutable @@default(false) approved: bool     // when true, the debit is drawn from an allowance granted to the transaction payer; the owner (accountId) need not sign
}

// One NFT transfer (single serial). NFTs are non-fungible so each serial moves exactly once
// per transaction.
type NftTransfer {
    @@immutable tokenId: Address                    // the NFT collection
    @@immutable serial: int64                       // the serial number being transferred
    @@immutable sender: AccountId                   // current owner
    @@immutable receiver: AccountId                 // new owner
    @@immutable @@default(false) approved: bool     // when true, the transfer is drawn from an NFT allowance granted to the transaction payer; the sender need not sign
}

// Atomic transfer of HBAR and/or fungible tokens and/or NFTs. All legs settle together: if any
// leg fails (insufficient balance, missing signature, frozen / KYC-blocked token account, ...)
// the entire transaction is rolled back.
@@finalType
TransferTransaction extends Transaction<TransferReceipt> {
    @@immutable @@default([]) hbarTransfers: list<HbarTransfer>
    @@immutable @@default([]) tokenTransfers: list<TokenTransfer>
    @@immutable @@default([]) nftTransfers: list<NftTransfer>
}

@@finalType
TransferReceipt extends Receipt {
}

// Allowance for HBAR. `amount` is absolute, not delta: re-approving overwrites the prior
// value; an amount of zero revokes the allowance.
type HbarAllowance {
    @@immutable ownerAccountId: AccountId           // the account granting the allowance (must sign the transaction)
    @@immutable spenderAccountId: AccountId         // the account that may spend on behalf of the owner
    @@immutable amount: NativeToken<ANY, ANY>       // upper bound the spender may transfer; 0 = revoke
}

// Allowance for a fungible token. `amount` is absolute, not delta: re-approving overwrites the
// prior value; an amount of zero revokes the allowance.
type TokenAllowance {
    @@immutable tokenId: Address                    // the fungible token the allowance applies to
    @@immutable ownerAccountId: AccountId           // the account granting the allowance (must sign the transaction)
    @@immutable spenderAccountId: AccountId         // the account that may spend on behalf of the owner
    @@immutable amount: int64                       // upper bound in the token's smallest unit; 0 = revoke
}

// Allowance for an NFT collection. Exactly one of `serials` or `approveAll` must be set:
//
// - `serials` grants the spender per-serial rights to the listed serials (additive across
//   multiple approve transactions).
// - `approveAll = true` grants the spender every current and future serial of `tokenId` the
//   owner holds; `approveAll = false` revokes a previously granted blanket grant for this
//   (owner, spender, tokenId) triple.
//
// `delegatingSpender` supports HIP-336: a spender that already holds an `approveAll` grant may
// delegate per-serial rights onward without involving the original owner. When set, the
// transaction must be signed by `delegatingSpender`, not by `ownerAccountId`.
@@oneOf(serials, approveAll)
type NftAllowance {
    @@immutable tokenId: Address
    @@immutable ownerAccountId: AccountId
    @@immutable spenderAccountId: AccountId
    @@immutable @@nullable serials: list<int64>
    @@immutable @@nullable approveAll: bool
    @@immutable @@nullable delegatingSpender: AccountId
}

// Grants delegated-spend rights from one or more owners. Per-leg semantics:
//
// - Every HBAR / fungible-token / per-serial NFT entry is independent — granting an NFT
//   allowance does not affect the owner's HBAR allowance to the same spender, etc.
// - The owner of each leg (`ownerAccountId`, or `delegatingSpender` for HIP-336 NFT
//   sub-approvals) must sign the transaction.
@@finalType
AccountAllowanceApproveTransaction extends Transaction<AccountAllowanceApproveReceipt> {
    @@immutable @@default([]) hbarAllowances: list<HbarAllowance>
    @@immutable @@default([]) tokenAllowances: list<TokenAllowance>
    @@immutable @@default([]) nftAllowances: list<NftAllowance>
}

@@finalType
AccountAllowanceApproveReceipt extends Receipt {
}

// One NFT-allowance revocation. Removes spender rights for the listed serials of `tokenId`
// regardless of which spender currently holds them; an NFT cannot be approved to more than one
// spender at a time, so the owner does not need to name the spender to revoke.
type NftAllowanceDeletion {
    @@immutable tokenId: Address                    // the NFT collection
    @@immutable ownerAccountId: AccountId           // the account revoking the allowance (must sign the transaction)
    @@immutable serials: list<int64>                // serials whose allowance is revoked
}

// Revokes previously granted NFT allowances. NFT-only — HBAR and fungible-token allowances are
// revoked by approving them with amount = 0, and blanket `approveAll` grants are revoked by
// approving with approveAll = false.
@@finalType
AccountAllowanceDeleteTransaction extends Transaction<AccountAllowanceDeleteReceipt> {
    @@immutable nftAllowanceDeletions: list<NftAllowanceDeletion>
}

@@finalType
AccountAllowanceDeleteReceipt extends Receipt {
}
```

## Examples

### Sender is the operator (single signature)

The operator moves value out of its own account. The operator signs as payer *and* as the
debited account in one step.

```
HieroClient client = ...;     // operator = alice
AccountId bob = ...;

Response<TransferReceipt> response = new TransferTransaction()
    .hbarTransfers([
        new HbarTransfer(client.operatorAccountId, NativeToken.of(-10, HBAR_UNIT)),
        new HbarTransfer(bob,                      NativeToken.of(+10, HBAR_UNIT)),
    ])
    .signWithOperatorAndSubmit(client);

response.queryReceipt();
```

### Sender is *not* the operator (multi-signature)

The operator pays the transaction fee, but Alice is the one being debited. Alice must add her
own signature; otherwise the network rejects the transaction with `INVALID_SIGNATURE`. The
flow goes through `PackedTransaction` so that both signatures end up on the same packed
payload.

```
HieroClient client = ...;     // operator is not Alice
Account alice = ...;          // separate account that holds the funds
AccountId bob = ...;

PackedTransaction<...> packed = new TransferTransaction()
    .hbarTransfers([
        new HbarTransfer(alice.accountId, NativeToken.of(-10, HBAR_UNIT)),
        new HbarTransfer(bob,             NativeToken.of(+10, HBAR_UNIT)),
    ])
    .signWithOperator(client)   // operator signs as payer
    .sign(alice);               // Alice signs to authorize her debit

Response<TransferReceipt> response = packed.submit(client);
```

### Atomic HBAR + fungible + NFT transfer

A single `TransferTransaction` settles HBAR, a fungible-token leg, and an NFT leg together. If
any leg fails (e.g. recipient not associated with the token) the entire transfer rolls back.
Alice is debited on every leg, so she must co-sign in addition to the operator.

```
PackedTransaction<...> packed = new TransferTransaction()
    .hbarTransfers([
        new HbarTransfer(alice.accountId, NativeToken.of(-5, HBAR_UNIT)),
        new HbarTransfer(bob,             NativeToken.of(+5, HBAR_UNIT)),
    ])
    .tokenTransfers([
        new TokenTransfer(usdcTokenId, alice.accountId, -100, 6, false),
        new TokenTransfer(usdcTokenId, bob,             +100, 6, false),
    ])
    .nftTransfers([
        new NftTransfer(nftCollectionId, /* serial */ 42L, alice.accountId, bob, false),
    ])
    .signWithOperator(client)
    .sign(alice);

packed.submit(client);
```

### Sponsored spend via allowance (`approved = true`)

Alice previously called `AccountAllowanceApproveTransaction` to grant the spender an HBAR
allowance. The spender (here: the operator) now drains it without involving Alice — Alice's
signature is *not* required because `approved = true` on the debit leg tells the network to
charge the spender's allowance instead.

```
new TransferTransaction()
    .hbarTransfers([
        new HbarTransfer(alice, NativeToken.of(-10, HBAR_UNIT), /* approved */ true),
        new HbarTransfer(bob,   NativeToken.of(+10, HBAR_UNIT)),
    ])
    .signWithOperatorAndSubmit(client);   // operator = spender, signs as payer + allowance holder
```

### Approve HBAR / fungible / NFT allowances in one transaction

The *owner* of each leg must sign. If the owner is the operator, `signWithOperatorAndSubmit`
is enough; otherwise the owner has to co-sign as in the transfer multi-signature example
above.

```
HieroClient client = ...;     // operator = alice (the owner of all legs below)

new AccountAllowanceApproveTransaction()
    .hbarAllowances([
        new HbarAllowance(client.operatorAccountId, spender, NativeToken.of(100, HBAR_UNIT)),
    ])
    .tokenAllowances([
        new TokenAllowance(usdcTokenId, client.operatorAccountId, spender, 1_000_000L),
    ])
    .nftAllowances([
        // Specific serials:
        new NftAllowance(nftCollectionId,   client.operatorAccountId, spender, [1L, 2L, 3L], null, null),
        // Or: blanket grant over every current and future serial:
        new NftAllowance(otherCollectionId, client.operatorAccountId, spender, null,         true, null),
    ])
    .signWithOperatorAndSubmit(client);
```

### Revoke allowances

```
// Revoke an HBAR or fungible allowance by approving zero (owner = operator = alice):
new AccountAllowanceApproveTransaction()
    .hbarAllowances([
        new HbarAllowance(client.operatorAccountId, spender, NativeToken.of(0, HBAR_UNIT)),
    ])
    .signWithOperatorAndSubmit(client);

// Revoke a blanket NFT grant by setting approveAll = false:
new AccountAllowanceApproveTransaction()
    .nftAllowances([
        new NftAllowance(otherCollectionId, client.operatorAccountId, spender, null, false, null),
    ])
    .signWithOperatorAndSubmit(client);

// Revoke specific NFT serials regardless of current spender:
new AccountAllowanceDeleteTransaction()
    .nftAllowanceDeletions([
        new NftAllowanceDeletion(nftCollectionId, client.operatorAccountId, [1L, 2L, 3L]),
    ])
    .signWithOperatorAndSubmit(client);
```

## Questions & Comments

- **`AccountAllowanceDeleteTransaction` is NFT-only by design.** HAPI exposes only an NFT
  removal list, and HBAR / fungible-token allowances are deleted by approving zero. We
  intentionally do not surface a fake "delete fungible allowance" affordance — it would have
  to translate to an approve(0) under the hood, hiding which signature is actually required
  (the *owner*, same as approve). <- That should be done in enterprise layer
