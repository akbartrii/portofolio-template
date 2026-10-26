# Repository Quality Checklist
Run through this before calling the project "done" and before pinning it on your profile.

## Basics
- [ ] Clear repo name, short description, topics/tags set
- [ ] README: problem, demo, features, architecture, how to run, how to test
- [ ] GIF/screenshot or video demo
- [ ] Live demo link (or a stated reason why not) + demo account / seed data
- [ ] LICENSE
- [ ] `.env.example` present; no secrets anywhere in git history
- [ ] Proper `.gitignore`

## Code & Architecture
- [ ] Consistent folder structure; linter & formatter enabled
- [ ] Architecture diagram (Mermaid) in README
- [ ] At least one ADR for an important design decision
- [ ] API documentation (OpenAPI/Swagger)
- [ ] Clean error handling & logging

## Testing
- [ ] Unit tests for core business logic
- [ ] Integration tests against a real database (Testcontainers/Docker)
- [ ] Tests for hard edge cases (concurrency, invalid input, upstream/AI failure)
- [ ] CI status badge (+ coverage if available)

## CI/CD & Ops
- [ ] GitHub Actions: lint → test → build on every PR
- [ ] Docker image builds; `docker compose up` works from a fresh clone
- [ ] Auto-deploy from `main` (if there is a live demo)
- [ ] Dependabot enabled; unused workflows/ecosystems removed

## History & Process
- [ ] Conventional Commits; small, meaningful commits
- [ ] Features developed via branch + Pull Request
- [ ] Issues/milestones reflect the roadmap
- [ ] Semver-tagged release + CHANGELOG

## Legal & Data
- [ ] Only public or synthetic data; data sources attributed
- [ ] No copyrighted assets (images, fonts, icons) without a clear license
- [ ] No employer-owned code or data

## Proof of Impact
- [ ] At least one measurable number in README (latency, accuracy, throughput)
- [ ] "Design Decisions & Trade-offs", "How I Used AI", and "What I Learned" sections filled in
