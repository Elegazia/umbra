# Architecture Decision Records

Each significant design decision in Umbra is recorded as an architecture decision record (ADR), giving the context, the options considered, the decision and its consequences. The format follows [MADR](https://adr.github.io/madr/), with an added Background section that explains the underlying concepts.

## Index

| ADR | Title | Status |
|---|---|---|
| [0001](0001-brokers-venues-and-data-sources.md) | Brokers, execution venues and market-data sources | Accepted |
| [0002](0002-instrument-identity-and-definition.md) | Instrument identity and definition schema | Accepted |
| [0003](0003-pre-commit-ruff-via-uv.md) | Run ruff in pre-commit through uv-managed local hooks | Accepted |
| 0004 | Storage format and query engine | Planned |
| 0005 | Numeric representation of prices and quantities | Planned |
| 0006 | Market-data record schemas and timestamp policy | Planned |
| 0007 | `DataSource` interface | Planned |
| [0008](0008-latency-target-and-deployment-model.md) | Latency target and deployment model | Accepted |

## Rules

1. Each ADR records one decision. If a document starts to decide a second thing, that decision gets its own ADR and the two link to each other.
2. An ADR moves from Proposed to Accepted, and later to Deprecated or "Superseded by NNNN". It becomes Accepted only after the deciders have reviewed the rationale.
3. Accepted ADRs are not changed in substance. Typos, wording and broken links can be fixed at any time. To change a decision, write a new ADR that supersedes the old one and update the status of both, so the history of the reasoning is kept.
4. Facts that change over time, such as licences, prices and API terms, carry an access date and are marked as primary (from the provider or regulator) or secondary (press, reviews, aggregators).
5. Context states what the system has to satisfy, not who is building it.
6. Each ADR says what would cause it to be revisited, in terms that can be checked.

## Template

```markdown
# ADR NNNN: <decision title>

- Status: Proposed | Accepted | Deprecated | Superseded by NNNN
- Date: YYYY-MM-DD (revised YYYY-MM-DD)
- Deciders: <names or roles>
- Related: <links to other ADRs>

## Context

<The problem, why it needs a decision now, and the constraints, stated as requirements.>

## Decision drivers

Hard constraints:

- D1. <...>

Preferences, in order:

- D2. <...>

## Considered options

### A. <name>

<How it works.>

- Good, because <...> (D1).
- Bad, because <...> (D2).

## Decision

Chosen: <option>.

<Why, in terms of the drivers, including the deciders' own reasoning.>

## Consequences

<What becomes easier, what becomes harder, risks and how they are handled, follow-up work.>

### Revisit when

- <checkable condition> (<what to reconsider>);

## Background

<The underlying concepts, explained from first principles, with references.>

## Sources

| Source | Type | Accessed |
|---|---|---|
| [Title](url) | Primary / Secondary | YYYY-MM-DD |
```
