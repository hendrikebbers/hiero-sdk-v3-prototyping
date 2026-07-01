# Testing Guideline

This document defines **how the V3 SDK specifications are tested** and **how tests are run**. It is the
counterpart to [`api-guideline.md`](api-guideline.md): where the API guideline defines the meta-language used to
*describe* the API, this document defines the meta-language and the platform used to *verify* it.

Read this before adding a `## Testing` section to any `spec/*.md` file.

## Philosophy

- **Every spec owns its tests.** A specification is only complete when it says how an implementation of it can be
  proven correct. Each spec file therefore carries a `## Testing` section that lists language-agnostic test
  scenarios for the types and methods it defines.
- **Tests are behavioural, not implementation-bound.** A test describes an observable behaviour of the public API
  (given a client, when a query is submitted, then a result is returned). It never references private types,
  internal classes, or a specific language binding. Each concrete SDK (Java, JS, Rust, …) translates the scenario
  into an idiomatic test in its own test framework, exactly as it translates the API schema into idiomatic code.
- **Tests run against a real network, not mocks.** The behaviours we specify are network behaviours (cost
  discovery, receipts, balances). Mocking the consensus node would only test our mock. Every scenario runs against
  a real, locally provisioned Hiero network — see below.

## Test platform: solo

[`solo`](https://solo.hiero.org) is the official tool for provisioning a local Hiero network on top of a local
Kubernetes cluster (Kind). It is the canonical test platform for the V3 SDK because it spins up a *real* consensus
node and mirror node with a single command, and funds default test accounts automatically.

The SDK models a solo network as a first-class network setting — see [`spec/base/solo.md`](../spec/base/solo.md):
the `solo` namespace exposes the `SOLO_IDENTIFIER` constant and `SoloNetworkSetting`, so tests resolve a solo
network through the same `NetworkSetting` mechanism as any other network.

### Prerequisites

| Requirement | Notes |
|-------------|-------|
| Node.js >= 22 | Required by solo (`node --version`). |
| solo CLI | macOS/Linux/WSL2: `brew install hiero-ledger/tools/solo` (bundles Node.js). Otherwise: `npm install -g @hiero-ledger/solo@latest`. Verify with `solo --version`. |
| Container runtime + memory | solo provisions kubectl, Helm and Kind automatically. A single node needs a few GB; multi-node needs ~16 GB RAM / 8 CPU. |

### Lifecycle

Tests are wrapped around a **deploy → run → destroy** lifecycle. Deployment is slow (it boots a Kubernetes
cluster), so it happens **once per test run**, not once per scenario.

```bash
# 1. Provision a single-node network (creates/connects a Kind cluster, funds default accounts)
solo one-shot single deploy

# 2. Run the SDK test suite for the language binding under test
#    (e.g. ./gradlew test, npm test, cargo test — defined by each implementation)

# 3. Tear the network down again
solo one-shot single destroy
```

For scenarios that require real multi-node consensus (e.g. behaviour that differs on a committee of nodes) use the
multi-node variant instead — it has higher resource requirements:

```bash
solo one-shot multi deploy --num-consensus-nodes 3
# ... run tests ...
solo one-shot multi destroy
```

Default service endpoints exposed by a one-shot deployment (solo 0.63+):

| Service | Endpoint |
|---------|----------|
| Consensus node (gRPC) | `localhost:35211` |
| Mirror Node REST | `http://localhost:38081` |
| Explorer UI | `http://localhost:38080` |
| JSON-RPC relay | `http://localhost:37546` |

### Obtaining a client (the solo fixture)

Every test starts from a `HieroClient` connected to the running solo network with a **funded operator account**.
`solo one-shot ... deploy` funds a set of default test accounts (genesis account `0.0.2` and friends); the exact
account id and private key of the deployment can be read from solo itself:

```bash
solo one-shot show deployment                       # deployment name
solo deployment config info --deployment <name>     # operator account id + keys
```

Expressed in the meta-language, the fixture every scenario assumes is:

```
// resolve the solo network registered under solo.SOLO_IDENTIFIER
NetworkSetting solo = NetworkSetting.getNetworkSetting(SOLO_IDENTIFIER);

// the operator account funded by `solo one-shot ... deploy`
Account operator = new Account(operatorAccountId, operatorPrivateKey);

HieroClient client = HieroClient.createClient(solo, operator);
```

Test scenarios refer to this simply as *"a `HieroClient` connected to a solo network with a funded operator
account"* and to `operator.accountId` as *"the operator's account id"*. Each language binding provides this fixture
once (a JUnit extension, a Vitest fixture, a Rust test harness, …) and reuses it across scenarios.

## Where tests live: the `## Testing` section

Each spec file gains a `## Testing` section. In the file skeleton it sits **after `## Examples` and before
`## Questions & Comments`**:

```
# Title
## Description
## API Schema
## Examples          (optional)
## Testing           (new)
## Questions & Comments
```

If a spec defines only pure data types with no observable network behaviour (e.g. a namespace of enums), its
`## Testing` section may simply state that there is nothing to test and why.

### Scenario format

Each scenario is a **Given / When / Then** block introduced by a stable, unique id. Ids are
`lowercase-kebab-case` and are prefixed with the namespace under test so they stay unique across the whole spec
tree, e.g. `queries.accounts/balance-of-operator-succeeds`.

- **Given** — the preconditions / fixture (usually the solo client, plus any entities the test must create first).
- **When** — the single action under test (build a request from the spec and submit it).
- **Then** — the observable, assertable outcome. "Completes without error" is a valid outcome; prefer also
  asserting something about the returned value.

Keep scenarios small and independent: one behaviour per scenario, no ordering dependency between scenarios. If a
scenario needs a precondition entity (a token, a topic, an account), create it in the **Given** using the relevant
transaction from its own spec — do not assume state left behind by another scenario.

### Example

```
### `example.namespace/thing-round-trips`

- **Given** a `HieroClient` connected to a solo network with a funded operator account.
- **When** a `ThingCreateTransaction` with a valid name is submitted and the returned `ThingId` is read back with
  a `ThingInfoQuery`.
- **Then** both calls complete without error and the returned `ThingInfo.name` equals the submitted name.
```

## Running the tests

- **Locally:** install solo, run the three-step lifecycle above around your language binding's test command.
- **CI:** the same lifecycle in a job that has a container runtime available. Deploy once at job start, run the
  suite, and always destroy in a cleanup step (even on failure) so runners are not left holding a Kind cluster.

## Conventions

- One behaviour per scenario; scenarios are independent and order-free.
- Scenario ids are stable — treat them like public API. Renaming an id is a breaking change to the test contract;
  add a new id and deprecate the old one instead.
- Prefer asserting on returned values over asserting only "no error".
- Create any required precondition state inside the scenario's **Given**; never rely on another scenario's
  side effects or on state baked into the solo image beyond the funded default accounts.
