# CLAUDE.md

Guidance for Claude Code (and other agents) when working in this repository.

## What this repository is

This is a **proof-of-concept (PoC) for the next generation ("V3") of client SDKs/libraries** for the
[Hiero](https://hiero.org) project under Linux Foundation Decentralized Trust (LFDT). Because Hedera and other
networks build on Hiero, V3 targets all of them — it is explicitly broader than just Hedera.

The current SDKs are all at version 2; **V3 is the codename for the new generation** that replaces them. V3 is
designed from scratch with **no backward-compatibility constraints** with V2.

**This repo contains specifications, not a shippable SDK.** There is no build system (no Maven/Gradle/npm/Cargo) and
nothing compiles. The API is defined once in a **language-agnostic meta-language** and is meant to be translated into
idiomatic implementations per language (Java, JavaScript/TypeScript, Go, Rust, Python, C++, Swift). The `.java`/`.js`
files under `guidelines/` are **illustrative reference snippets**, not a buildable module.

## Repository structure

```
guidelines/
  api-guideline.md              # THE meta-language: syntax + cross-cutting API best practices. Read this first.
  api-best-practices-java.md    # How meta-language concepts map to idiomatic Java
  api-best-practices-rust.md    # ... Rust
  api-best-practices-js.md      # ... JavaScript
  java-files/                   # Illustrative Java reference implementations (streaming, thread-safety, etc.)
  js-files/                     # Illustrative JS reference snippets

spec/                           # The actual V3 public-API specifications, written in the meta-language
  base/                         # Foundational namespaces shared by everything
    common.md (common)          #   Page<$$T> and other shared types
    ledger.md (ledger)          #   Ledger, Address, ConsensusNode, MirrorNode
    ledger-config.md (ledger.config)
    keys.md (keys, keys.io)     #   Keys + key import/export (PKCS#8, SPKI, DER, PEM)
    native-token.md (hbar)      #   Hbar native token + exchange rate
    grpc.md (grpc)
    proto.md (proto)
  consensus-node-client/        # Talking to the consensus node
    client.md (consensusnode.client)              # HieroClient, Account, TransactionSigner
    transactions.md (consensusnode.transactions)  # Transaction<$$Receipt>, Response, Receipt, Record
    transactions-accounts.md (consensusnode.transactions.accounts)
    transactions-spi.md (consensusnode.transactions.spi)   # SPI for custom services/transaction types
    proto.md / proto-accounts.md (consensusnode.proto[.account])
  mirror-node-client/           # Querying the Hiero Mirror Node REST API
    mirror-node.md (mirrornode) # MirrorNodeClient with per-domain repositories
    mirror-node-{account,contract,network,nft,token,topic,transaction,common}.md
  enterprise/                   # High-level "service" layer on top of the raw SDK for app/framework integration
    service.md (enterprise.service)
    service-account.md (enterprise.service.account)
    service-contract.md (enterprise.service.contract)
```

### How the layers relate

- **`base`** — primitives every layer depends on (`ledger`, `keys`, `hbar`, `common`, `proto`, `grpc`).
- **`consensusnode.*`** — the low-level client: build/sign/execute transactions against the consensus node. The
  `spi` namespace exists because the consensus node is service-oriented and supports *custom* services and transaction
  types, so the SDK must be extensible (hence `TransactionStatus` is an abstraction with an `int32` code, not a closed
  enum).
- **`mirrornode.*`** — read-side: query historical/state data from a Mirror Node over REST, returning paginated
  `Page<$$T>` results.
- **`enterprise.service.*`** — a higher-level convenience/service layer designed for easy use and deep framework
  integration (cf. Hiero Enterprise Java / JS). Provides factory methods today; real deployments use dependency
  injection.

## The spec meta-language (most important convention)

**Before writing or editing any `spec/*.md`, read `guidelines/api-guideline.md`** — it is the source of truth. Key
points to keep specs valid and consistent:

- Each spec file follows the same skeleton: `# Title` → `## Description` → `## API Schema` (a fenced code block) →
  optional `## Examples` → `## Questions & Comments`.
- The `## API Schema` block opens with `namespace <name>` and, if it uses other namespaces, `requires a, b, c`.
- **Cross-namespace references must be qualified** (`ledger.Address`, `common.Page`); same-namespace types use the
  simple name.
- **Naming:** Types `PascalCase`; fields & methods `lowerCamelCase`; enum values `UPPER_SNAKE_CASE`; namespaces
  `lowerCamelCase` (dots for sub-namespaces, no hyphens); constants `UPPER_SNAKE_CASE`; `@@throws` error ids
  `lowercase-kebab-case` (e.g. `not-found-error`).
- **Generic type parameters are prefixed with `$$`** (e.g. `Page<$$T>`, `Transaction<$$Receipt extends Receipt>`).
- **Annotations use a `@@` prefix.** Common ones: `@@immutable`, `@@nullable`, `@@default(v)`, `@@static`, `@@async`,
  `@@streaming`, `@@throws(...)`, `@@threadSafe[(group)]`, validation (`@@min/@@max/@@minLength/...`), and type-level
  `@@oneOf(...)`, `@@oneOrNoneOf(...)`, `@@finalType`.
- Abstract types use `abstraction`; inheritance uses `extends`; enums use `enum`.

### Cross-cutting design principles (enforce these in specs and reviews)

- **Immutability first:** annotate fields `@@immutable` by default; introduce mutability only with a clear reason.
- **Never define nullable collections** (`list`/`set`/`map`) — return empty instead of null.
- **Avoid `ANY` as a standalone type** — prefer a generic `$$T`, a concrete base type, a `@@oneOf` union, or `bytes`
  with a documented schema. (`ANY` as a *wildcard type argument* like `ContractParam<ANY>` is fine.)
- `@@async` returns a future/promise; `@@streaming` returns a pull-based async stream of items (`streamResult<TYPE>`
  for per-item errors). They are mutually exclusive.

## How to make changes

- **Editing/adding a spec:** keep the section skeleton, declare the `namespace`/`requires`, qualify cross-namespace
  types, and follow the naming + annotation rules above. Match the style of neighboring spec files.
- **Open design questions** belong under each file's `## Questions & Comments` (often attributed to a GitHub handle).
  Don't silently resolve them; surface them.
- **Language best-practice docs** (`api-best-practices-*.md`) describe how a meta-language concept maps to one
  language — when you add a new meta-language feature, consider whether each language guide needs a mapping (the
  guideline lists per-language mappings for varargs, wildcards, streaming cancellation, `streamResult`, etc.).
- Note: `api-guideline.md` references a `proposals/` folder, but in this repo the specs live under `spec/`.
- Some language guides referenced by `api-guideline.md` (cpp, ts, python, go, swift) do not exist yet — that's
  expected; only Java, Rust, and JS guides are present so far.

## Relevant skills

When working here, these skills are particularly useful:

- **`java-api-design`** and **`java-best-pratices`** — when reasoning about API surface, SPI, interfaces vs. records,
  module boundaries, breaking changes.
- **`modern-java`** — when writing or reviewing the `guidelines/java-files/` reference snippets (use modern idioms:
  records, sealed types, pattern matching, etc.).
- **`adr-create`** — when a genuine architectural decision is made, record it as an ADR.
- **`hiero-info`** / **`hedera-info`** — background when writing prose about Hiero/Hedera concepts.
- **`grill-me`** — to stress-test an API design before committing to it.

## Audience & language

The repository (README, guidelines, specs) is written in **English** for an international open-source audience — keep
new docs in English. Commit messages and PR descriptions in English as well.