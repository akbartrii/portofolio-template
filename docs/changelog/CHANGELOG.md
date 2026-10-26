# Changelog

All notable changes to this project are documented here.
Format based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow [Semantic Versioning](https://semver.org/).

## [Unreleased]
### Added
- Initial project setup
- Project documentation: PRD, business rules, modules, workflows, architecture, API spec, data model, testing strategy, operations, roadmap
- Architecture decision records 0002-0005
- Verified BMKG `adm4` catalog (15 cities)
- Project README and environment template for the BMKG weather aggregator

### Changed
- Scoped the project to TypeScript/Node with Node 24 LTS
- Replaced database-oriented env template with application/BMKG/cache settings
- Reorganized `docs/` into `analysis`, `architecture`, `changelog`, `operations`, `product`, and `prompts`
- Moved `CHANGELOG.md` to `docs/changelog/CHANGELOG.md`
- Added `docs/README.md` as the documentation index

### Removed
- Java CI workflow
- Docker workflow (until a Dockerfile exists)
