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
