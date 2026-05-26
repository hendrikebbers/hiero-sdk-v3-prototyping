# hiero-sdk-v3-prototyping

A prototyping ground for the **next generation of client SDKs ("V3")** for the
[Hiero](https://hiero.org) project under Linux Foundation Decentralized Trust (LFDT). Since Hedera and other networks
build on Hiero, V3 is designed for the whole ecosystem — not just Hedera.

This repository does **not** contain a shippable SDK. It holds the **language-agnostic API specification** for V3,
the conventions used to write that specification, and the per-language best-practice guides that translate it into
idiomatic implementations.

## Why V3?

The current Hiero SDKs were originally built for Hedera and the Hedera network. With the move of the Hedera codebase
to Hiero, the project has become much broader in scope. The existing SDKs (all at version 2) carry limitations that no
longer align with where the project is heading:

- **Scoped to Hedera** — they were designed for the Hedera network and don't reflect Hiero's broader scope.
- **Language idioms not fully embraced** — some SDKs feel foreign to developers experienced in that language, causing
  avoidable friction.
- **Outdated architecture** — they have not kept pace with the modularization of the consensus node and support for
  custom services and transactions.
- **Developer experience** — developers deserve SDKs that are intuitive, consistent across languages, and aligned with
  modern API design principles.

## Vision

V3 is an opportunity to design the ideal SDK from the ground up, without backward-compatibility constraints. The goal
is a public API that is:

- **Language-idiomatic** — each SDK feels natural in its language (generics, type inference, idiomatic error handling).
- **Consistent across languages** — concrete APIs adapt per language, but the structure, concepts, and workflows stay
  consistent.
- **Future-proof** — the architecture accommodates Hiero's evolution (custom services, modular consensus nodes, new
  transaction types) without breaking changes.
- **Accessible and well-documented** — clear guidelines, comprehensive docs, and example-driven design.
- **Broader than Hedera** — works with any Hiero-based network.

Beyond the consensus node, V3 also targets client APIs for the **Hiero Mirror Node** and **Hiero Block Node**, and is
designed with **framework integration** (e.g. Hiero Enterprise Java / JS) in mind from the start.

## Approach

- **Language-agnostic API definition** — the public API is specified once in a
  [meta-language](guidelines/api-guideline.md) and translated into concrete implementations per language.
- **Prototype-driven design** — rather than designing purely on paper, we draft the API and validate it through
  prototypes to ensure the design works in practice.
- **No backward-compatibility constraints** — V3 is designed as an ideal API. Migration paths from V2 will be
  addressed once concrete MVPs are available and the design has stabilized.

## Repository layout

| Path | Contents |
|------|----------|
| [`guidelines/api-guideline.md`](guidelines/api-guideline.md) | The meta-language: syntax + cross-cutting API best practices. **Start here.** |
| [`guidelines/api-best-practices-java.md`](guidelines/api-best-practices-java.md) | How the meta-language maps to idiomatic Java |
| [`guidelines/api-best-practices-rust.md`](guidelines/api-best-practices-rust.md) | ... Rust |
| [`guidelines/api-best-practices-js.md`](guidelines/api-best-practices-js.md) | ... JavaScript |
| `guidelines/java-files/`, `guidelines/js-files/` | Illustrative reference snippets (not a buildable module) |
| `spec/base/` | Foundational namespaces: `ledger`, `keys`, `hbar`, `common`, `proto`, `grpc` |
| `spec/consensus-node-client/` | Low-level client: build, sign, and execute transactions (incl. an SPI for custom services) |
| `spec/mirror-node-client/` | Querying the Hiero Mirror Node REST API |
| `spec/enterprise/` | High-level service layer for easy use and framework integration |

## Writing specifications

Every spec under `spec/` is written in the meta-language defined in
[`guidelines/api-guideline.md`](guidelines/api-guideline.md). In short:

- Each file follows the skeleton: `## Description` → `## API Schema` → optional `## Examples` →
  `## Questions & Comments`.
- The API schema declares a `namespace` and imports the types it uses with `requires {Type} from namespace`
  (one statement per source namespace); imported types are then referenced by their simple name (e.g. `Address`).
- Fields are **immutable by default** (`@@immutable`); collections are never nullable; the `ANY` top type is avoided
  as a standalone type.
- Naming: types `PascalCase`, fields/methods `lowerCamelCase`, enum values & constants `UPPER_SNAKE_CASE`, namespaces
  `lowerCamelCase`, error ids `lowercase-kebab-case`. Generic type parameters are prefixed with `$$`.

See [`CLAUDE.md`](CLAUDE.md) for a fuller orientation aimed at contributors and AI coding agents.

## Target languages

V3 covers all Hiero SDKs: **Java, JavaScript / TypeScript, Go, Rust, Python, C++, and Swift.**

## Status

Early prototype. The API surface, namespaces, and open questions (tracked in each spec's
`## Questions & Comments` section) are still evolving and subject to change.