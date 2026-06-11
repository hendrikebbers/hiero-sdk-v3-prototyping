# TCK Ideas for V3

> **Status:** Working document, not a decision. Two architectures are actively in play
> (hand-written Java tests vs. codegen from a meta-language scenario corpus). The decision
> will be recorded as an ADR once a small validation spike has narrowed it down. This file
> preserves the analysis so the spike has context to start from.

## 1. Goals and constraints

### 1.1 What the TCK has to do

- Validate that every conforming V3 SDK implements the **public API** defined in
  [`spec/`](spec/) identically. Conformance is the only goal — internal-implementation tests
  belong in the individual SDK repos.
- Cover the **edge-case surface that V3 has accumulated**: n-of-n signing on `FileCreate`,
  `0.0.0`-as-clear-sentinel ([`ZERO_ADDRESS`](spec/base/ledger.md)),
  `@@oneOf(serials, approveAll)` on NFT allowances, chunked-append leakiness on
  `FileAppend`, second-precision `seconds` typing. These are the places where two
  conforming-on-paper SDKs could silently diverge.
- Run **real network calls** — no mocks, no fake state. Tests have to exercise actual HAPI
  semantics, otherwise they catch nothing.
- Surface failures **per-test** with clear attribution: which SDK, which scenario, which
  parameter, which assertion.

### 1.2 What is settled (and unaffected by the open architecture question)

These ground rules apply regardless of Option A or Option B below:

- **Test network:** Solo ([`hiero-ledger/solo`](https://github.com/hiero-ledger/solo)),
  the official Hiero CLI for standalone test-network deployment. The spec already reserves
  the [`solo` namespace](spec/base/solo.md) and a `SOLO_IDENTIFIER` constant.
- **Solo lifecycle:** one Solo instance per TCK run. Started in `@BeforeAll` (or the
  codegen equivalent), shut down in `@AfterAll`. Snapshot/restore between tests is an
  additive future optimisation — deferred.
- **Test isolation:** by **discipline**, not by snapshot. Each test allocates its own
  Accounts, Topics, Files via helper factories. No test depends on state another test
  created. Tests don't assume ordering. The first confirmed flake caused by cross-test
  state contamination is the trigger to reconsider snapshots.
- **Repository:** a **new** repo, `hiero-sdk-v3-tck`, separate from
  `hiero-sdk-v3-prototyping` and from the existing V2 TCK. Clean break.
- **Public-API testing only:** tests call the SDK from the outside; no internal-only
  hooks.
- **Per-SDK execution:** each SDK must be able to be exercised individually. If the chosen
  architecture eliminates per-SDK drivers (Option B), it does so because each SDK runs
  natively-generated tests; the *requirement* of per-SDK execution stays.

### 1.3 What is still being explored

- **Test definition language** — Java (with classic JUnit) vs. an extension of the V3
  meta-language with codegen per target language.
- **Wire protocol between TCK and SDK** — only relevant under Option A; vanishes under
  Option B. Decision blocked until A/B is settled.

## 2. Why this is worth analysing carefully

V3 is intentionally spec-first and multi-language: Java, TypeScript, Go, Rust, Python, C++,
Swift ([`CLAUDE.md`](CLAUDE.md)). A TCK choice locks in *7×* the long-term maintenance
cost in the wrong direction. Two structural traps:

1. **V2 TCK lessons**:
   [`hiero-ledger/hiero-sdk-tck`](https://github.com/hiero-ledger/hiero-sdk-tck) uses
   JSON-RPC drivers per SDK with a TypeScript test runner. JSON loses type fidelity at the
   wire. TypeScript can't syntactically distinguish synchronous from asynchronous calls —
   every Promise-returning call needs `await`, no matter the API's actual contract. V3
   makes sync/async explicit in the spec via `@@async` / `@@streaming`, so the TCK has to
   be able to express the distinction the spec promises.
2. **Driver maintenance scales with SDK count.** Whatever shape we pick, the
   "per-language work" item determines whether the TCK can keep up as new SDKs land.

## 3. The two contenders

### Option A — Hand-written Java tests with classic JUnit, plus per-SDK driver

Tests live as Java source in the TCK repo, using JUnit Jupiter (`@Test`,
`@ParameterizedTest`, `@MethodSource`, `@Nested`, `@Tag`, `@TestFactory`).
[JUnit User Guide](https://junit.org/junit5/docs/current/user-guide/) provides everything
needed: parameterisation, dynamic tests, hierarchical grouping, CI slicing, reporting.

Each SDK ships a **driver** — a process that the TCK calls into. The driver receives
serialized method-call instructions over a wire protocol (JSON-RPC, gRPC, or
meta-language-derived schema — undecided, blocked until A/B is settled) and executes them
against its native V3 SDK against Solo.

```java
@Tck
class TransferTransactionTests extends TckBase {

    @Nested class HbarTransfers {

        @Test
        void simpleTransferBetweenTwoAccounts() {
            var alice = newAccount(hbar(100));
            var bob   = newAccount();

            var receipt = new TransferTransaction()
                .hbarTransfers(List.of(
                    HbarTransfer.debit (alice, hbar(10)),
                    HbarTransfer.credit(bob,   hbar(10))
                ))
                .sign(alice)
                .submit()
                .queryReceipt();

            assertThat(receipt.status).isEqualTo(SUCCESS);
            assertThat(balanceOf(alice)).isEqualTo(hbar(90));
            assertThat(balanceOf(bob)).isEqualTo(hbar(10));
        }

        @ParameterizedTest(name = "amount={0}")
        @ValueSource(longs = { 1, 100, 1_000_000, Long.MAX_VALUE / 1_000_000 })
        void transferAcrossAmountRange(long amountTinybars) { … }

        @ParameterizedTest(name = "keyType={0}")
        @EnumSource(KeyType.class)
        void transferWithDifferentOperatorKeyTypes(KeyType keyType) { … }

        @Test @Tag("multi-sig")
        void allowanceBackedTransfer_payerSignsForOwner() { … }
    }

    @Nested class TokenTransfers { … }
    @Nested class NftTransfers { … }
}
```

The synchronous-looking style works because **virtual threads**
([JEP 444, finalised in JDK 21](https://openjdk.org/jeps/444)) let test bodies call
blocking SDK methods cheaply, while still leaving `CompletableFuture` available when the
async shape itself is under test. Structured concurrency
([JEP 505 — fifth preview in JDK 25](https://openjdk.org/jeps/505); sixth preview as
[JEP 525 in JDK 26](https://openjdk.org/jeps/525)) is still preview and is **not** relied
on.

### Option B — Extend the meta-language to define test scenarios; codegen native tests per SDK

The same meta-language that defines `spec/` types and method signatures (with `@@async`,
`@@streaming`, `@@oneOf`, etc.) is extended with a small set of scenario constructs.
Scenarios live in the TCK repo as meta-language source. A codegen produces idiomatic
native test source per target language: JUnit for Java, Vitest for TypeScript, `cargo test`
for Rust, Go `testing`, pytest, Swift XCTest. Each SDK compiles and runs **native** tests
against its own V3 SDK against Solo.

**The decisive consequence:** under Option B, **there is no per-SDK driver**. There is no
wire protocol. There is no opaque-handle bookkeeping. Each SDK team consumes generated
tests in their language's native test framework and runs them as part of their build.
Seven driver codebases collapse into one codegen pipeline.

#### 3.1 Concrete syntax sketches

The minimal extension: four new constructs. `@@scenario`, `let`, `assert`, `@@concurrent`.
Plus optional `@@parameters`, `@@stream`, `@@property` for specialisations.

**Simple scenario** (sequential `@@async` calls implicitly awaited):

```
namespace tck.consensusnode.transfers
requires {TransferTransaction, HbarTransfer} from consensusnode.transactions.accounts
requires {SUCCESS} from consensusnode.transactions
requires {hbar} from nativeToken.hbar
requires {newAccount, balanceOf} from tck.fixtures

@@scenario hbarTransferBetweenTwoAccounts {
    let alice = newAccount(initialBalance: hbar(100))
    let bob   = newAccount()

    let receipt = new TransferTransaction()
        .hbarTransfers([
            HbarTransfer{accountId: alice.accountId, amount: hbar(-10)},
            HbarTransfer{accountId: bob.accountId,   amount: hbar(10)}
        ])
        .sign(alice)
        .submit()                       // @@async → implicit await
        .queryReceipt()                 // @@async → implicit await

    assert receipt.status == SUCCESS
    assert balanceOf(alice) == hbar(90)
    assert balanceOf(bob)   == hbar(10)
}
```

**Async — implicit by default, explicit when needed.** The `@@async` annotation on the
spec method is the source of truth. In scenario context every `@@async` call is implicitly
awaited; that covers 95% of tests. The `@@concurrent` block opts into real concurrency
when the test cares:

```
@@scenario twoParallelTransfersFromSameAccount {
    let alice = newAccount(hbar(100))
    let bob   = newAccount()
    let carol = newAccount()

    let (r1, r2) = @@concurrent {
        new TransferTransaction()
            .hbarTransfers([
                HbarTransfer{accountId: alice.accountId, amount: hbar(-10)},
                HbarTransfer{accountId: bob.accountId,   amount: hbar(10)}
            ])
            .sign(alice).submit().queryReceipt(),

        new TransferTransaction()
            .hbarTransfers([
                HbarTransfer{accountId: alice.accountId, amount: hbar(-5)},
                HbarTransfer{accountId: carol.accountId, amount: hbar(5)}
            ])
            .sign(alice).submit().queryReceipt()
    }

    assert r1.status == SUCCESS
    assert r2.status == SUCCESS
}
```

Codegen per language:

| Sprache | sequential `@@async`                    | `@@concurrent`                                     |
|---------|-----------------------------------------|----------------------------------------------------|
| Java    | blocking call on virtual thread         | `CompletableFuture.allOf(...)` (or `StructuredTaskScope` once JEP 505 finalises) |
| TS      | `await`                                 | `Promise.all([...])`                               |
| Rust    | `.await`                                | `tokio::join!(a, b)`                               |
| Go      | blocking call (every call is blocking)  | `errgroup.Group`                                   |
| Python  | `await`                                 | `asyncio.gather(a, b)`                             |
| Swift   | `await`                                 | `async let`                                        |
| C++     | blocking (with coroutine stub)          | `std::experimental::when_all`                      |

**Streaming — explicit, pull-based block:**

```
@@scenario topicReceivesAllSubmittedMessages {
    let topic = newTopic()
    let payloads = ["alpha", "beta", "gamma"]

    @@stream subscription = mirror.topic(topic.topicId).subscribe() {
        @@concurrent {
            for payload in payloads {
                new TopicMessageSubmitTransaction()
                    .topicId(topic.topicId)
                    .message(payload.bytes)
                    .signAndSubmit()
            }
        }

        for expected in payloads {
            let received = next(subscription)
            assert received.message == expected.bytes
        }
    }
    // subscription auto-closes on block exit (try-with-resources semantics)
}
```

`next(stream)` is the only new primitive. Per-language mapping:

| Sprache | `next(stream)`                                                  |
|---------|-----------------------------------------------------------------|
| Java    | `subscription.next()` on a blocking `Flow.Subscriber` wrapper   |
| TS      | `await subscription.next()` (AsyncIterator)                     |
| Rust    | `subscription.next().await` (`futures::Stream`)                 |
| Go      | `<-subscription` (channel)                                      |
| Python  | `await subscription.__anext__()`                                |

Negative tests (no message arrives) use a separate `expectIdle(stream, duration)` helper.

**Parameterisation — static cross-product via annotation:**

```
@@parameters {
    amount: int64 from [1, 100, 1_000_000, 8_000_000_000]
    keyType: KeyType from [ED25519, ECDSA_SECP256K1]
}
@@scenario hbarTransferAcrossAmountsAndKeys(amount: int64, keyType: KeyType) {
    let alice = newAccount(hbar(amount * 2), keyType: keyType)
    let bob   = newAccount()

    let receipt = new TransferTransaction()
        .hbarTransfers([
            HbarTransfer{accountId: alice.accountId, amount: hbar(-amount)},
            HbarTransfer{accountId: bob.accountId,   amount: hbar(amount)}
        ])
        .sign(alice).submit().queryReceipt()

    assert receipt.status == SUCCESS
}
```

Codegen produces the cross-product as the native idiom: `@ParameterizedTest` +
`@MethodSource` in Java, `it.each([...])` in TS, `#[rstest] #[case(...)]` in Rust,
table-driven `t.Run` in Go, `@pytest.mark.parametrize` in Python.

**Property-based testing — the hard case.** Simple generators (uniform numeric ranges,
enum sampling) work declaratively:

```
@@property amountInValidRange {
    given amount: int64 in (0, MAX_HBAR_SUPPLY) generated by uniformLong
    runs 100
    @@scenario { ... }
}
```

Complex generators (e.g., "generate a `Key`, then a `KeyList` of 1–5 such generators
nested") collapse into either a generator-function escape hatch (must also go through
codegen — awkward) or stay out of the corpus and remain hand-written per SDK. Pragmatic
cut: **simple property-based in the corpus; complex property-based stays sprach-native.**
Complex property-based is anyway secondary to the TCK's main job (conformance) — those
tend to be robustness/fuzz tests an SDK team runs independently.

#### 3.2 When does Option B win?

Two conditions must hold for codegen to beat hand-written Java:

1. **The V3 spec stabilises.** As long as weekly spec refactors keep happening (the last
   ten conversation rounds: `seconds`, `ZERO_ADDRESS`, signing-table fixes, the `Key`
   sum-type gap), every refactor would cost two weeks of codegen work. Once V3 freezes
   the spec, the codegen tooling becomes cheap to maintain.
2. **The team is willing to swap driver investment for codegen investment.** Both are
   non-trivial. Driver work is N×codebase ongoing maintenance; codegen is 1×codebase
   ongoing maintenance. At seven SDKs the long-term tradeoff favours codegen by a
   factor of ~7.

If only the second condition holds (we'd accept the upfront cost but the spec is still
moving), Option B becomes a permanent chase. If only the first holds (spec is frozen but
the team has no appetite for codegen tooling), Option A is fine. The case for Option B is
when **both** hold.

## 4. Honest comparison — Option A vs Option B

| Aspekt                                          | Option A (Java + JUnit + Driver)        | Option B (DSL + Codegen)                                     |
|-------------------------------------------------|-----------------------------------------|--------------------------------------------------------------|
| Initial build effort                            | small (skeleton + helpers)              | **large** (codegen per language + DSL parser)                |
| Maintenance per new SDK language                | **driver write + maintain**             | only codegen backend (one-time then maintained)              |
| Wire-protocol complexity                        | open (separate ADR needed)              | **eliminated entirely**                                      |
| Type fidelity in tests                          | good in Java, weaker across wire        | **perfect in every language**                                |
| Async / streaming clarity                       | good in Java, depends on wire           | **native per language**                                      |
| Debuggability on test failure                   | good for Java, mixed for others         | **good in every language (native stacktrace)**               |
| New failure class                               | n/a                                     | **codegen bug vs SDK bug — extra diagnostic burden**         |
| Dynamic / complex property-based tests          | trivial with jqwik                      | **only simple cases in corpus**                              |
| Refactor safety on spec change                  | TCK breaks at `javac`                   | **codegen breaks, all languages break synchronously**        |
| Risk while spec still moves                     | medium                                  | **high** (codegen chases spec)                               |
| Long-term maintenance cost at N=7 SDKs          | **N drivers × ongoing**                 | one codegen × ongoing                                        |

## 5. Other alternatives — considered and rejected

### V2-style: TypeScript runner + JSON-RPC drivers

Re-targeting the existing V2 TCK to V3. **Rejected** as a starting point:

- TypeScript can't syntactically distinguish synchronous from asynchronous calls — every
  Promise-returning call needs `await`, regardless of the API's intent. V3 makes
  sync/async explicit in the spec via `@@async`/`@@streaming`, so the TCK must be able to
  express the distinction.
- JSON at the wire loses generics. Parameterised types
  (`Transaction<$$Receipt>`, `PaidQuery<$$Result>`, `Page<$$T>`) get flattened.
- Re-targeting V2 code locks in V2's driver shape just when V3 gives us a chance to
  redesign.

### JEP 512 (compact source files / instance main methods), one scenario per file

[JEP 512](https://openjdk.org/jeps/512), finalised in JDK 25 after a four-round preview
chain (JEP 445 → 463 → 477 → 495), makes a single-scenario file extremely terse (`void
main()` no class declaration). **Rejected** for the test corpus:

- Parameterisation has no first-class story. Every viable workaround reinvents what
  JUnit's `@ParameterizedTest` / `@TestFactory` provides as a mature, IDE-integrated
  mechanism.
- Per-permutation failure isolation, per-test setup/teardown, tagging, Surefire/Gradle
  reporting all have to be hand-rolled.
- "One file per scenario" elegance dissolves at hundreds of files — `@Nested`-structured
  classes are easier to navigate.

JEP 512 stays useful for ad-hoc demonstration snippets (spec's `Examples` sections), but
is the wrong tool for a maintained TCK.

### Gherkin / declarative scenarios with per-SDK step harnesses

**Rejected.** Gherkin is intentionally programming-language-weak (no variable binding, no
async composition, no test-as-value). Step implementations end up being most of the
work, and they live in each SDK's native language anyway — identical in scope to a driver
under Option A or generated tests under Option B, but with a constrained scenario DSL on
top.

### GraalVM polyglot in-process

**Rejected.** GraalVM's polyglot runtime doesn't host Go, Rust, or Swift. Half of the V3
target platforms can't participate. Out.

## 6. Proposed next step — a scoped spike

Before either committing to Option A or revising the direction toward Option B, a small
**validation spike** sharpens the question with concrete evidence rather than continued
abstract argument:

1. **Write a DSL mini-spec** (1–2 pages). Exactly the five constructs above
   (`@@scenario`, `let`, `assert`, `@@concurrent`, `@@stream`) and nothing more.
2. **Express three real scenarios** in the DSL:
   - simple HBAR transfer
   - multi-sig transfer (`signWithOperator(client).sign(...)` chain)
   - topic-message streaming consumer
   Covers sync, multi-step, concurrent, and streaming.
3. **Write a codegen backend for two languages**: Java + Rust. Two deliberately different
   async models (virtual threads vs. `async fn`/`tokio`). If both back-ends produce
   idiomatic native code from the same DSL source, the concept is viable.
4. **Evaluate on three questions**:
   - Is the DSL more pleasant for a test author than hand-written Java JUnit?
   - Is the generated Java code as good as hand-written JUnit?
   - Is the generated Rust code idiomatic (cargo test, async-style)?

Feasible in ~1–2 weeks. No TCK repo bootstrap, no driver design, no wire-protocol
decision needed beforehand. The spike answers a binary question: codegen viable here, or
codegen too costly for this spec's stability profile.

**If the spike succeeds:** ADR records Option B as the chosen direction. ADR-for-wire-protocol
becomes unnecessary.

**If the spike fails:** ADR records Option A as the chosen direction. The next ADR
addresses the wire protocol.

## 7. Open questions

Beyond the A/B decision itself:

- **Wire protocol** (Option A only): JSON-RPC (V2 status quo) vs gRPC with
  meta-language-derived schema vs another option. Blocked until A/B is settled.
- **Snapshot/restore between tests**: deferred. Re-open when first state-contamination
  flake appears, or when CI runtime grows unacceptable.
- **Property-based scope**: simple-only in the corpus is the proposed cut; complex
  generators stay sprach-native. To be confirmed once the spike has a property-based
  example.
- **Supported-network-versions matrix**: per TCK release, a `supported-network-versions.md`
  lists which Solo / consensus-node versions were validated against. Format TBD.
- **TCK ownership**: which team(s) own the TCK repo, the codegen (if Option B), and the
  release cadence? Open organisational question.

## 8. Assumptions still to confirm

- **Solo cold-start time** of ≈30 s is anecdotal. Will be measured during the first CI
  setup and documented. The "one instance per run" plan is robust to a wide range (60 s,
  120 s) — only a multi-minute startup would force a rethink.
- **"Every SDK team has enough Java fluency to contribute scenarios."** True if Option A
  is chosen and ownership concentrates on the Java-fluent team. Under Option B every team
  works in their native language, sidestepping this concern entirely.

## References

- V3 spec under test:
  [`spec/`](spec/), [`CLAUDE.md`](CLAUDE.md), [`missing-features.md`](missing-features.md),
  [`spec/base/solo.md`](spec/base/solo.md), [`spec/base/ledger.md`](spec/base/ledger.md),
  [`guidelines/api-guideline.md`](guidelines/api-guideline.md).
- V2 TCK for comparison:
  [`hiero-ledger/hiero-sdk-tck`](https://github.com/hiero-ledger/hiero-sdk-tck) —
  JSON-RPC test runner + driver-per-SDK pattern.
- Solo network:
  [`hiero-ledger/solo`](https://github.com/hiero-ledger/solo) — "An opinionated CLI tool
  to deploy and manage standalone test networks." Documentation site at
  [`solo-docs`](https://github.com/hiero-ledger/solo-docs).
- Java language features cited:
  - [JEP 512: Compact Source Files and Instance Main Methods](https://openjdk.org/jeps/512)
    — finalised in JDK 25, descends from preview chain JEP 445 → 463 → 477 → 495.
  - [JEP 444: Virtual Threads](https://openjdk.org/jeps/444) — finalised in JDK 21.
  - [JEP 505: Structured Concurrency (Fifth Preview)](https://openjdk.org/jeps/505) — still
    preview as of JDK 25; 6th preview tracked under
    [JEP 525](https://openjdk.org/jeps/525) for JDK 26. Not relied on.
- JUnit Jupiter:
  [JUnit 5 User Guide](https://junit.org/junit5/docs/current/user-guide/) —
  `@ParameterizedTest`, `@MethodSource`, `@Nested`, `@Tag`, `@TestFactory` references.
