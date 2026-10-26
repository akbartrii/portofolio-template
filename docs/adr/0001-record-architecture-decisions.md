# ADR-0001: Record architecture decisions

- **Status:** Accepted
- **Date:** YYYY-MM-DD

## Context
Design decisions in this project should be traceable, so readers (and future me) understand not just *what* was built but *why*.

## Decision
Significant decisions are recorded as short Architecture Decision Records in `docs/adr/`, using `0000-template.md`.

## Consequences
Each important choice (database, message broker, locking strategy, etc.) gets one short file. This adds a few minutes per decision and makes trade-offs explicit.
