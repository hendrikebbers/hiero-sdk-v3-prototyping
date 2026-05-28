# Service API

The service layer is a layer on top of the classical SDK functionalities (that exist in V2 and V3). The idea is to
make it easier to use the SDK and to allow a better integration in enterprise applications. Every concrete service
(`AccountService`, `FungibleTokenService`, `NftService`, `TopicService`, `FileService`, `SmartContractService`) is
constructed from a `Session` defined in this namespace.

## Description

A `Session` is the **consistency boundary** of the enterprise service layer. It bundles three concerns that every
concrete service shares:

1. **Network configuration** — the `NetworkSetting` describing the target ledger and any tunables.
2. **Operator credentials** — the `Account` used to pay for transactions, plus an optional `TransactionSigner` if the
   private material lives outside the process (HSM, KMS, wallet).
3. **Read-your-writes consistency across services** — every successful write executed through a service of the
   session advances a *high-water-mark* (the `consensusTimestamp` of the receipt). Before any service of the same
   session performs a Mirror-Node-backed read, it waits until the Mirror Node has data at or beyond that high-water-
   mark. This ensures that, within one session, `nftService.transferNft(...)` followed by
   `accountService.findById(...)` reflects the balance change, even though the two calls hit different back-ends
   (consensus node vs. mirror node) with different propagation delays.

### Why the session must span all services

The high-water-mark is a **global ledger property**, not per service type. A `SmartContractService.callContractFunction`
can change HBAR balances, NFT ownership, token supply, and topic state in a single transaction. Splitting the
consistency scope per service type would let a follow-up `accountService.findById` see a stale state right after the
contract call. Putting every service of a workflow into the same session is therefore the only correct granularity.

### Streaming reads

Streaming operations such as `TopicService.subscribe(topicId)` do **not** participate in the wait — they are live
subscriptions, not snapshot reads, and applying the wait would defeat their purpose. Every non-streaming Mirror-Node-
backed read does participate.

### Multiple sessions

Sessions are independent of each other. Using two sessions in the same process (e.g. one per request, one per
background worker, one per user, one per operator account) creates two isolated consistency boundaries. There is no
implicit cross-session synchronisation; a write in session `A` does not block a read in session `B`.

### Read-only workflows

If a session never performs a write, its high-water-mark stays `null` and no Mirror Node read ever blocks. Pure read
workloads pay no consistency cost.

### Opt-out

The service layer offers **no toggle to disable the wait**. Applications that need raw throughput should go directly
to the lower-level `mirrornode.*` and `consensusnode.*` APIs, which expose the same operations without the
consistency guarantee.

### Threading

`Session` is `@@threadSafe`: the high-water-mark is monotonically advanced (newer timestamps win) and can be read and
advanced concurrently from any number of threads or asynchronous tasks.

### Scoping in framework integrations

The correct lifetime of a `Session` is **one workflow**, not one process. A workflow is whatever logical unit a write
and its follow-up reads belong to: a single HTTP request, a single batch-job iteration, a single scheduled task run,
a single message processed by a worker. Sharing one `Session` across the whole server is wrong — it would couple
unrelated workflows (user A's write would block user B's reads) and undermine the isolation property documented
above.

How that maps to common framework integrations:

- **Spring (servlet stack):** `Session` becomes a `@RequestScope` bean (with `ScopedProxyMode.TARGET_CLASS`). The
  concrete `*Service` beans stay singletons and hold the scoped-proxy, so developers can `@Autowired` services into
  any controller or service bean without thinking about lifecycle. Inside the request, every service sees the same
  underlying session instance.
- **Spring (reactive / WebFlux):** request-scope is not thread-bound, so the session lives in the Reactor
  `ContextView` instead. The adapter is responsible for putting a fresh `Session` into the context at the start of
  the exchange and for forwarding it across `flatMap` boundaries.
- **Kotlin coroutines / structured concurrency:** the session is a `CoroutineContext.Element` propagated by the
  language's own context-propagation rules.
- **Non-HTTP workloads (`@Scheduled`, `@Async`, Spring Batch, Kafka listeners, CLI tools):** there is no built-in
  scope. Either define a custom scope (`@JobScope`, `@StepScope`, thread-bound scope) or create and close a session
  manually around the unit of work (`try`/`with`/`use`).
- **Plain SDK usage / scripts / tests:** one `Session` per logical operation, created via `createSession(...)` and
  discarded afterwards.

A framework adapter is therefore responsible for two things:

1. Picking the right scope abstraction (request, exchange, job, message, …) and binding `Session` to it.
2. Hiding the scope from the developer so that `@Autowired NftService` (or its language-equivalent) "just works".

### Distributed workflows (multiple processes)

When one workflow spans multiple processes — e.g. service A submits a transaction and synchronously calls service B
which then queries the Mirror Node — the high-water-mark must travel with the call. Today this is not supported and
must be solved at a higher layer (e.g. by sleeping, by waiting on an event, or by always going through service A for
reads). A future iteration of `Session` is expected to add an opaque consistency-token export/import pair so that an
HTTP/gRPC interceptor (analogous to W3C Trace Context propagation) can forward the token across process boundaries.
See the open question below.

## API Schema

```
namespace enterprise.service
requires {NetworkSetting} from ledger.config
requires {Account, TransactionSigner} from consensusnode.client

@@threadSafe
abstraction Session {

    // Block until the Mirror Node has caught up to the current high-water-mark (with timeout).
    // Returns immediately when the high-water-mark is null. Throws service-error on timeout.
    @@async @@throws(service-error) void awaitConsistency()
}

// Default factory: the session signs every transaction with the operator account's private key.
@@static Session createSession(networkSettings: NetworkSetting, operatorAccount: Account)

// Factory for setups where signing happens out of process (HSM, KMS, wallet).
@@static Session createSession(networkSettings: NetworkSetting, operatorAccount: Account, transactionSigner: TransactionSigner)
```

## Questions & Comments

- **Cross-process consistency-token propagation.** Should `Session` expose `exportConsistencyToken(): @@nullable
  string` and `importConsistencyToken(token: string)` so that a framework adapter can forward the high-water-mark
  across HTTP/gRPC boundaries via a header (e.g. `X-Hiero-Consistency`)? The token would be opaque so the
  underlying representation (currently just a `zonedDateTime`) can evolve. Out of scope for the first iteration but
  required as soon as workflows span multiple processes — see "Distributed workflows" above.
- Should `awaitConsistency()` accept an explicit timeout override per call, or only be configured at session
  creation? Currently the wait/timeout policy is an implementation detail of the session.
- Is `service-error` enough on `awaitConsistency()`, or do we want a distinct `consistency-timeout-error` so callers
  can branch on it? Today the surrounding services already throw `service-error` for everything, so introducing a
  new id would only pay off if applications regularly need to recover from this specific failure mode.
