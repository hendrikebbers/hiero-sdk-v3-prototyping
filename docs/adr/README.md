# Architecture Decision Records

This directory collects the Architecture Decision Records (ADRs) for the V3 SDK
specification. Each ADR captures **one** architecturally significant decision
with its motivation, the alternatives considered, and the trade-offs accepted.

ADRs are sequentially numbered (`NNNN-kebab-case-title.md`) and are never
deleted — superseded ADRs link forward to the decision that replaces them.

## Index

| # | Title | Status |
|---|---|---|
| [0001](0001-transaction-and-packed-transaction-split.md) | Split the transaction API into `Transaction` and `PackedTransaction` | Accepted |
| [0002](0002-defer-paid-query-payer-customization.md) | Defer payer customization for paid queries | Accepted |
| [0003](0003-three-level-address-hierarchy-with-nullability-narrowing.md) | Three-level address hierarchy with nullability narrowing | Accepted |
| [0004](0004-authority-authorization-sum-type.md) | Model HAPI authorization keys as an `Authority` sum type | Proposed |
