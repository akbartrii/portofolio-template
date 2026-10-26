# Contributing

## Branches
`feat/<short-description>`, `fix/...`, `chore/...`, `docs/...`

## Commits
This repo follows [Conventional Commits](https://www.conventionalcommits.org/):
```
feat(scope): add something
fix(scope): correct something
refactor | test | docs | ci | chore | perf
```
Keep commits small and focused. One logical change per commit.

## Documentation
Docs live under `docs/` grouped by category:

```
docs/
  analysis/       testing, quality checklist
  architecture/   overview, modules, workflows, API, data model, ADR
  changelog/      CHANGELOG.md
  operations/     runbook
  product/        PRD, business rules, roadmap, design spec
  prompts/        AI prompt log
```

Update `docs/README.md` and any linking document when adding or moving a doc.

## Pull Requests
1. Open or link an issue.
2. Create a branch from `main`.
3. Open a PR and complete the checklist.
4. CI must pass before merging.
