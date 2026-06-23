# File Transactions API

## Description

The file service stores arbitrary byte content on the consensus node, addressed by an
`Address` (file id). Write access is gated by a *key list*: every key in the file's current
key list must sign any modification (`FileAppend`, `FileUpdate`, `FileDelete`) — this is an
**n-of-n** requirement, not m-of-n. The classic uses are smart-contract bytecode, system
parameters, and signed binary artifacts that need on-ledger provenance.

A file's lifecycle is:

```
  create ── (append | update)* ── delete
```

- `FileCreate` allocates a new file id, sets the initial key list, optional memo, expiration
  time, and the first slice of content.
- `FileAppend` appends bytes to the end of the file.
- `FileUpdate` replaces any subset of the metadata (keys, memo, expiration) and optionally
  *replaces* the entire content (not appends — see the field comment on `contents`).
- `FileDelete` deletes the file. The file id is not reusable; reads against a deleted file
  return `FILE_DELETED`.

### Chunked appends

The consensus node imposes a maximum body size per transaction. A `FileAppend` whose
`contents` exceeds that limit cannot be sent as a single transaction; the SDK splits it into
N consecutive `FileAppend` transactions on the same `fileId` and submits them in order. From
the caller's perspective this remains a single `Transaction.signWithOperator…` call — the SDK
hides the split. The receipt surfaced back to the caller is the receipt of the *last* chunk.

This is a leaky abstraction: each underlying chunk is a separate consensus transaction with
its own fee and its own `TransactionId`. The current model intentionally keeps the surface
simple; surfacing per-chunk receipts is deferred (see *Questions & Comments*). The same
chunking concept applies to `TopicMessageSubmit` and other HAPI transactions that ride on the
consensus-node body size limit.

### Signing requirements (file)

| Transaction         | Signers required                                                                          |
|---------------------|-------------------------------------------------------------------------------------------|
| `FileCreate`        | the *payer* **and** every key in the new `key` authorization (anti-spoofing: cannot bind another party's key as a file modifier without their consent) |
| `FileAppend`        | the *payer* **and** every key in the file's current `key` authorization                   |
| `FileUpdate`        | the *payer* **and** every key in the *current* `key` authorization; if `key` changes, every key in the *new* authorization must sign as well |
| `FileDelete`        | the *payer* **and** every key in the file's current key list                              |

When the operator is also the sole key on the file, `signWithOperatorAndSubmit(client)` is
enough. Multi-key files require the multi-signature flow shown in
[`transactions-accounts.md`](transactions-accounts.md): build → `signWithOperator(client)` →
`.sign(...)` per custodian → `submit(client)`.

## API Schema

```
namespace consensusnode.transactions.files
requires {Address} from ledger
requires {Authority} from authority
requires {Receipt, Transaction} from consensusnode.transactions

@@finalType
FileCreateTransaction extends Transaction<FileCreateReceipt> {
    @@immutable @@default([]) contents: bytes                  // initial content; further bytes via FileAppend
    @@immutable @@nullable authority: Authority                    // write-access authorization; an all-must-sign rule is an AuthorityList
    @@immutable @@nullable expirationTime: zonedDateTime       // when the file expires; SDK default if unset
    @@immutable @@nullable fileMemo: string                    // short, human-readable label (max 100 chars)
}

@@finalType
FileCreateReceipt extends Receipt {
    @@immutable fileId: Address                                // id of the newly created file
}

// Appends `contents` to the end of the file. The SDK transparently splits this into multiple
// consensus transactions when `contents` exceeds the single-transaction body limit.
@@finalType
FileAppendTransaction extends Transaction<FileAppendReceipt> {
    @@immutable fileId: Address
    @@immutable contents: bytes                                // bytes to append; large payloads are chunked by the SDK
}

@@finalType
FileAppendReceipt extends Receipt {
}

// Updates one or more of a file's mutable fields. Every nullable field is "leave unchanged"
// when null. Setting `contents` REPLACES the file contents in full — to add bytes without
// dropping the existing content, use `FileAppendTransaction` instead.
@@finalType
FileUpdateTransaction extends Transaction<FileUpdateReceipt> {
    @@immutable fileId: Address
    @@immutable @@nullable contents: bytes                     // when set, replaces the entire file contents
    @@immutable @@nullable authority: Authority                    // write-access authorization; an all-must-sign rule is an AuthorityList
    @@immutable @@nullable expirationTime: zonedDateTime       // when set, extends (or sets) the expiration time
    @@immutable @@nullable fileMemo: string                    // when set, replaces the memo
}

@@finalType
FileUpdateReceipt extends Receipt {
}

@@finalType
FileDeleteTransaction extends Transaction<FileDeleteReceipt> {
    @@immutable fileId: Address
}

@@finalType
FileDeleteReceipt extends Receipt {
}
```

## Examples

### Create a file (operator is the sole key)

```
HieroClient client = ...;          // operator's key will also gate the file

Response<FileCreateReceipt> response = new FileCreateTransaction()
    .contents("hello world".bytes)
    .authority(Authority.of(client.operatorPublicKey))
    .fileMemo("greeting")
    .signWithOperatorAndSubmit(client);

Address fileId = response.queryReceipt().fileId;
```

### Append (transparently chunked when needed)

`largeBlob` may be hundreds of kilobytes; the SDK splits it into multiple `FileAppend`
consensus transactions internally and returns the final receipt.

```
new FileAppendTransaction()
    .fileId(fileId)
    .contents(largeBlob)
    .signWithOperatorAndSubmit(client);
```

### Update — replace contents, rotate keys (multi-signature)

`contents` is replaced (not appended). Rotating the authorization to an all-must-sign pair
requires signatures from *both* the current and the new keys, in addition to the payer.

```
PackedTransaction<...> packed = new FileUpdateTransaction()
    .fileId(fileId)
    .contents("v2 payload".bytes)
    .authority(Authority.of(Authority.of(newKeyA), Authority.of(newKeyB)))   // all-must-sign AuthorityList
    .signWithOperator(client)        // payer + current key (operator)
    .sign(newSignerA)                // new key
    .sign(newSignerB);               // new key

packed.submit(client);
```

### Delete

```
new FileDeleteTransaction()
    .fileId(fileId)
    .signWithOperatorAndSubmit(client);
```

## Questions & Comments

- **`key` is a single `Authority`.** HAPI's `FileCreateTransactionBody.keys` is a single
  `KeyList` — one authorization requirement whose entries must *all* sign to modify the file —
  so V3 models it as one `key: Authority` rather than a `list`. The all-must-sign rule is an
  `AuthorityList` with `threshold == children.size()`, and because `Authority` is recursive
  each entry may itself be a single key, a contract, or a nested threshold — e.g.
  `Authority.of(Authority.of(alice), Authority.of(2, Authority.of(bob), Authority.of(carol), Authority.ofContract(dao)))`.
  The `Authority` authorization type is introduced in
  [ADR-0004](../../docs/adr/0004-authority-authorization-sum-type.md) and specified in
  [`authority.md`](../base/authority.md), resolving the former `list<PublicKey>` placeholder.

- **Chunked appends are not modelled at the type level.** `FileAppendTransaction.contents`
  accepts the full byte payload and relies on the SDK to split it into multiple consensus
  transactions when the body exceeds the single-tx size limit. The model returns a single
  `Response<FileAppendReceipt>` — there is no first-class way for advanced callers to inspect
  per-chunk receipts, control chunk size, or recover from a partially-applied append (where
  chunk K succeeded but chunk K+1 timed out). Once HIP-1300 (jumbo transactions, section
  1.11 of `missing-features.md`) is specified, the choice between "explicit chunk control"
  and "always pretend it's one transaction" needs a decision; the same question applies to
  `TopicMessageSubmit`.

- **`FileUpdate.contents` replaces, not appends.** This mirrors HAPI semantics but is a
  common foot-gun: callers expecting "update" to mean "merge" will silently drop the rest of
  the file. The field comment calls this out; consider whether the API should also expose a
  dedicated `replaceContents(bytes)` / `appendContents(bytes)` split at the language-binding
  level to make intent explicit.

- **`FileCreate` does not set `shard` / `realm`.** They are inherited from the
  client's configured network. Once persistent shard / realm
  (HIP-1299, section 5.2 in `missing-features.md`) is added to the client, no per-transaction
  shard / realm fields are needed here either.

- **No `autoRenewPeriod` field.** Unlike accounts, files in HAPI do not currently carry an
  auto-renew account or auto-renew period — expiration is set explicitly. Should the v3
  surface anticipate a future HIP that adds auto-renew to files, or stay aligned with the
  current protocol shape?
