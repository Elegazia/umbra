# Architecture Decision Records

Each significant design decision in Umbra is recorded as an ADR: the context, the options considered, the decision and its consequences. The format follows [MADR](https://adr.github.io/madr/), with an added **Background** section explaining the underlying concepts.

## Index

| ADR | Title | Status |
|---|---|---|
| [0001](0001-brokers-venues-and-data-sources.md) | Brokers, execution venues and market-data sources | Accepted |
| [0002](0002-instrument-identity-and-definition.md) | Instrument identity and definition schema | Proposed |
| [0003](0003-pre-commit-ruff-via-uv.md) | Run ruff in pre-commit through uv-managed local hooks | Accepted |
| 0004 | Storage format and query engine | Planned |
| 0005 | Numeric representation of prices and quantities | Planned |
| 0006 | Market-data record schemas and timestamp policy | Planned |
| 0007 | `DataSource` interface | Planned |

## Rules

1. **One decision per ADR.** If a document starts deciding a second thing, that decision gets its own ADR and the two link to each other.
2. **Status lifecycle:** `Proposed` → `Accepted` → `Deprecated` or `Superseded by NNNN`. An ADR becomes `Accepted` only when the rationale has been reviewed by the deciders.
3. **Accepted ADRs are immutable in substance.** Fix typos and broken links freely. To change a decision, write a new ADR that supersedes the old one and update both statuses, so the reasoning history stays intact.
4. **Evidence is dated.** Time-sensitive facts (licences, prices, API terms) carry an access date and are marked as primary (the provider or regulator) or secondary (press, reviews, aggregators).
5. **Constraints are requirements.** Context states what the system must satisfy, not who is building it.
6. **Revisit triggers are measurable.** Each ADR names the conditions that would reopen it.

## Template

```markdown
# ADR NNNN: <decision title>

- **Status:** Proposed | Accepted | Deprecated | Superseded by NNNN
- **Date:** YYYY-MM-DD (revised YYYY-MM-DD)
- **Deciders:** <names or roles>
- **Related:** <links to other ADRs>

## Context

<The problem, why it needs a decision now, and the constraints as requirements.>

## Decision drivers

Ranked. Hard constraints (must hold) first, then preferences (trade-offs).

1. **D1 (hard): …**
2. **D2: …**

## Options considered

### Option A: <name>

<How it works.>

| Driver | Assessment |
|---|---|
| D1 | … (evidence: [source], accessed YYYY-MM-DD) |

- **Benefits:** …
- **Costs:** …

## Decision

<The chosen option.>

**Rationale:** <Why, tied to the drivers.>

## Consequences

- **Easier:** …
- **Harder:** …
- **Risks and mitigations:** …
- **Follow-up work:** …

### Revisit triggers

- <Measurable condition> → <what to reconsider>

## Background

<The underlying concepts from first principles, with references.>

## Sources

| Source | Type | Accessed |
|---|---|---|
| [Title](url) | Primary / Secondary | YYYY-MM-DD |
```
