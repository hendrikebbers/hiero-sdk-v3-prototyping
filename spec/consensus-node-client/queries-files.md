# File Queries API

This namespace defines the read-side counterpart to the file transactions in
[`transactions-files.md`](transactions-files.md). Both queries are *paid*: the consensus node
charges a fee for serving file content and metadata.

## Description

Two queries are exposed:

- **`FileContentsQuery`** extends `PaidQuery<FileContents>` — returns the raw byte payload of
  a file. The price is roughly proportional to file size; callers that fetch large files
  should set `maxQueryPayment` to bound the spend.

- **`FileInfoQuery`** extends `PaidQuery<FileInfo>` — returns the file's metadata snapshot:
  size, expiration time, deletion flag, current key list, and memo. Does not return the
  content (use `FileContentsQuery` for that).

Both extend `PaidQuery`, so they inherit the `maxQueryPayment` knob, the `getCost(client)`
cost-discovery method, and the `PaidQueryResponse<...>` envelope. See
[`queries.md`](queries.md) for the full payment / envelope semantics.

A query against a deleted file does not fail — `FileInfoQuery` returns a `FileInfo` with
`deleted = true` (and most other fields zeroed / cleared), and `FileContentsQuery` returns an
empty `contents` byte array. Callers should branch on `FileInfo.deleted` rather than treat an
empty `contents` payload as authoritative evidence of deletion.

## API Schema

```
namespace consensusnode.queries.files
requires {Address} from ledger
requires {PublicKey} from keys
requires {PaidQuery} from consensusnode.queries

// Raw byte payload of a file. Returned by FileContentsQuery.
type FileContents {
    @@immutable fileId: Address                       // the file this snapshot belongs to
    @@immutable contents: bytes                       // full file content; empty for a deleted file
}

// Paid query for the raw byte content of a file. The fee scales with file size; bound
// the spend with `maxQueryPayment` for large files.
@@finalType
FileContentsQuery extends PaidQuery<FileContents> {
    @@immutable fileId: Address
}

// Full file metadata snapshot. Returned by FileInfoQuery.
type FileInfo {
    @@immutable fileId: Address
    @@immutable size: int64                            // current file size in bytes
    @@immutable expirationTime: zonedDateTime          // when the file expires
    @@immutable deleted: bool                          // true if the file has been deleted
    @@immutable keys: list<PublicKey>                  // current write-access key list (n-of-n on modification)
    @@immutable @@nullable fileMemo: string
}

// Paid query for the metadata of a file. Does not return the file's byte content — use
// FileContentsQuery for that.
@@finalType
FileInfoQuery extends PaidQuery<FileInfo> {
    @@immutable fileId: Address
}
```

## Examples

### Read file contents (paid)

```
HieroClient client = ...;

FileContents contents = new FileContentsQuery()
    .fileId(fileId)
    .maxQueryPayment(NativeToken.of(...))       // bound spend for large files
    .submit(client)
    .value;

bytes payload = contents.contents;
```

### Read file metadata, branch on deleted

```
FileInfo info = new FileInfoQuery()
    .fileId(fileId)
    .submit(client)
    .value;

if (info.deleted) {
    return;                                     // file no longer exists; other fields not meaningful
}

int64 sizeInBytes = info.size;
zonedDateTime expiresAt = info.expirationTime;
```

### Confirm price up-front for a large file

```
PaidQuery<FileContents> query = new FileContentsQuery().fileId(fileId);

NativeToken<ANY, ANY> quotedCost = query.getCost(client);
// show quotedCost to the user, wait for approval, then:

FileContents contents = query.submit(client).value;
```

`getCost(client)` is non-binding — the price on the subsequent `submit(client)` may differ;
combine it with `maxQueryPayment` for a hard ceiling. See
[`queries.md`](queries.md#examples) for the full pattern.

## Questions & Comments

- **`FileInfo.keys` is `list<PublicKey>`, not a `Key` tree.** Same placeholder typing as on
  the write side; see [`transactions-files.md`](transactions-files.md) *Questions & Comments*
  for why a flat list of public keys cannot represent the recursive `Key` sum type that HAPI
  actually returns (entries may be `KeyList`, `ThresholdKey`, `ContractID`, …). Tracked in
  [`missing-features.md`](../../missing-features.md) section 2.2 — the `Key` row is a
  prerequisite for retyping this field.

- **No `ledgerId` on `FileInfo`.** HAPI's `FileGetInfoResponse.FileInfo` carries a
  `ledger_id`, but the V3 `FileInfo` (and `AccountInfo` in
  [`queries-accounts.md`](queries-accounts.md)) deliberately omit it: the caller already
  knows which ledger they queried through their `HieroClient`, and ledger-origin metadata
  belongs on the response envelope, not duplicated on every typed payload. The corresponding
  envelope-level question is opened in [`queries.md`](queries.md) *Questions & Comments*.

- **`FileInfo` carries no `autoRenewAccount` / `autoRenewPeriod`.** Matches HAPI today —
  files do not auto-renew the way accounts and tokens do. The same open question as on the
  write side (see [`transactions-files.md`](transactions-files.md) *Questions & Comments*)
  applies: stay aligned with the current protocol or anticipate a future HIP that introduces
  auto-renew for files.
