# Using `hiero-sdk-tck` to test the V3 SDKs

Analysis of how the [hiero-ledger/hiero-sdk-tck](https://github.com/hiero-ledger/hiero-sdk-tck)
project can be used and extended to automate testing of a V3 SDK / client
implementation in any of the supported languages (Java, JavaScript/TypeScript,
Go, Rust, Python, C++, Swift).

## What the TCK is today

A **Technology Compatibility Kit** that ships as a TypeScript/Mocha test driver.
It treats the SDK-under-test as a black box reached via **JSON-RPC over HTTP**
(default `http://localhost:8544/`):

- The test driver sends JSON-RPC requests (`setup`, `createAccount`,
  `getAccountInfo`, …) to a server **the SDK author writes** in their own
  language.
- The driver checks the JSON response and **cross-verifies against a Mirror
  Node** (today using a pinned static JS SDK as the verifier).
- Tests are grouped by consensus-node service: `crypto-service`,
  `token-service`, `file-service`, `topic-service`, `contract-service`,
  `schedule-service`, `node-service`, `common`. Each group has spec docs under
  `docs/test-specifications/` and a `TestSpecificationsTemplate.md` for new
  ones.
- Reference TCK servers already exist for **Go** (`hiero-sdk-go/tck`) and
  **Java**.

## How V3's four layers map onto the TCK

| V3 layer | Fit with current TCK | What to do |
|---|---|---|
| **`consensusnode.*`** (`client`, `transactions`, `transactions.accounts`, `proto`, `spi`) | **Direct fit** — this is exactly what the TCK already exercises. | Each V3 language ships a thin **TCK adapter**: a JSON-RPC server that maps each TCK method to a V3 `Transaction<$$Receipt>` flow. Reuse the existing spec suite unchanged. |
| **`base.*`** (`keys`, `nativeToken`/`hbar`, `ledger`, `common`, `proto`, `grpc`) | **Implicit fit only** — keys/hbar/ledger are used by every consensus call but never tested as first-class. | Add new JSON-RPC namespaces (`keys.*`, `nativeToken.*`, `ledger.*`) so import/export round-trips (PKCS#8/SPKI/DER/PEM), `hbar` arithmetic, and `Ledger` resolution are tested without going through a transaction. |
| **`mirrornode.*`** | **Conflicting fit** — the TCK currently uses Mirror Node as the *oracle*, not as a system under test. | Add a `mirror.*` namespace and decouple verification: the SDK-under-test's mirror client should be the thing being tested, with an independent verifier (raw REST call or pinned other-language SDK). Cover `Page<$$T>` pagination and per-domain repositories explicitly. |
| **`enterprise.service.*`** | **No fit today.** | Add `enterprise.*` JSON-RPC namespaces (`enterprise.service.account.createAccount`, etc.) so the high-level facade is tested through the same harness — useful because it validates that DI / session / read-after-write behavior is consistent across languages. |

## How to use it for any supported language

The intended workflow is **one TCK adapter per language**, not one TCK per
language:

1. **Write a small JSON-RPC server in each V3 language** (Java, JS/TS, Go, Rust,
   Python, C++, Swift) that:
   - Implements `setup` (operator credentials, network selection — already
     abstracted in V3 via `Ledger` / `LedgerConfig`, so trivial to wire).
   - Dispatches each TCK method to the corresponding V3 call.
   - Returns `NOT_IMPLEMENTED` for methods the language hasn't built yet (the
     TCK tolerates this).
2. **Run the unchanged TCK Docker image** in CI against the adapter (the TCK
   already ships a `Dockerfile` and a `Taskfile.yaml`).
3. **Cover network variants** by re-running with `solo`, `hiero-local-node`, and
   `testnet` configurations — V3's `ledger` namespace makes this a setup-time
   switch.

## How to extend it for V3

The cleanest extension is **spec-driven generation**, because V3's defining
property is that the API is written in the meta-language under `spec/`:

- Define a **canonical mapping**: each meta-language method → JSON-RPC method
  name (e.g. `consensusnode.transactions.accounts.createAccount`). Annotations
  like `@@async` / `@@streaming` map to single response vs. streamed batches;
  `@@throws(id)` maps to JSON-RPC error codes using the kebab-case ids already
  defined in the specs (`not-found-error`, …), avoiding the current Hedera-string
  error codes that bake in Hedera-isms.
- **Auto-generate adapter scaffolding** from the spec in each language (similar
  to how `mirror-node.yaml` already generates TS models in the TCK). This keeps
  the JSON-RPC surface in lock-step with the spec and gives every language the
  same set of methods for free.
- Contribute new spec docs under `docs/test-specifications/` for each new
  namespace (`base/`, `mirror/`, `enterprise/`), following
  `TestSpecificationsTemplate.md`.
- For the streaming primitives in V3 (`@@streaming` → `streamResult<TYPE>`),
  extend the JSON-RPC convention with a notification-stream pattern (JSON-RPC
  2.0 supports server→client notifications), and add cancellation semantics
  matching the language guides.

## Caveats worth flagging up front

- The current TCK protocol carries **Hedera-specific status strings**
  (`REQUESTED_NUM_AUTOMATIC_ASSOCIATIONS_EXCEEDS_ASSOCIATION_LIMIT`, etc.). V3
  deliberately abstracts the native token and ledger, so before reusing the TCK
  a mapping is needed from V3's `@@throws` ids to those strings, or an
  extension that lets V3 errors round-trip unmodified.
- The static-JS-SDK Mirror Node oracle has to go (or be made
  language-agnostic) if the V3 `mirrornode.*` client is to be a real
  system-under-test rather than trusted infrastructure.
- The TCK has no concept today of the **SPI / custom transaction types** that
  V3's `consensusnode.transactions.spi` exposes — testing extensibility
  requires a new spec category.

## TL;DR

Use the TCK as-is for `consensusnode.*` by writing one JSON-RPC adapter per V3
language; extend it with new method namespaces and spec docs to also cover
`base`, `mirrornode`, and `enterprise`; and drive both the adapter and the
JSON-RPC method list from the meta-language specs so all seven languages
converge on the same compatibility surface.

## Sources

- [hiero-ledger/hiero-sdk-tck (GitHub)](https://github.com/hiero-ledger/hiero-sdk-tck)
- [Hiero SDK TCK docs portal](https://hiero-ledger.github.io/hiero-sdk-tck/)
- [hiero-sdk-go/tck (Go reference adapter)](https://pkg.go.dev/github.com/hiero-ledger/hiero-sdk-go/tck)
- [hiero-sdk-go/tck/methods](https://pkg.go.dev/github.com/hiero-ledger/hiero-sdk-go/tck/methods)
- [JSON-RPC 2.0 spec](https://www.jsonrpc.org/)
