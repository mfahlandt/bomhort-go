# Changelog

All notable changes are documented here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versions follow [semantic versioning](https://semver.org) (see [docs/COMPATIBILITY.md](docs/COMPATIBILITY.md)).

## [Unreleased]

Initial version, extracted from [VEXViper](https://github.com/seebom-labs/VEXViper)'s `internal/bomhort` and extended to the complete BOMHort API. Targets **BOMHort 0.8** (versions follow BOMHort, see [docs/COMPATIBILITY.md](docs/COMPATIBILITY.md)); works against 0.7 for everything 0.7 has.

### Added

- Client for every public BOMHort api-gateway route: health probes; SBOM listing, lookup, detail, findings, dependencies, licenses, VEX statements, download (buffered and streaming); uploads of SBOM and VEX documents with ownership, tags, parent and source attribution; source-attribution `PATCH`; fleet-wide vulnerabilities, VEX statements and affected projects; projects (incl. detail, SBOMs, vulnerabilities, packages, parent groups), tags, clusters, namespaces, fleet; dashboard, dependency and version-skew statistics; search, package search/detail, archived packages; license compliance, violations, exceptions and policy.
- BOMHort 0.8.0 API: `?scope=` filter (`VulnerabilityOptions`, `VulnerabilityListOptions`, `ScopeRoot`/`ScopeDirect`/`ScopeTransitive`/`ScopeUnknown`) on all finding lists (#444); `VulnerabilityReportCSV` (streamed `GET /api/v1/reports/vulnerabilities?format=csv`, #266); `LicenseSources` (`GET /api/v1/licenses/sources`); `LicenseViolationSummaries` (`?summary=true`).
- BOMHort 0.8.0 fields: `CVEID`/`Aliases` on `Vulnerability`, `VEXStatement` and `SearchVulnerability` (#412); `DependencyScope`/`DependencyDepth` on `Vulnerability`, `DependencyNode` and `AffectedProject` (#444); `DependencyNode.LicenseSource`; `PackageScopes` on `SBOMLicense` and `LicenseViolation`; `LicenseViolation.UnapprovedCount`.
- `bomhorttest`: `?scope=` filtering (400 on invalid values), `404` for the dependencies of an unknown SBOM (BOMHort 0.8 behaviour), raw `[]byte` bodies via `Handle` for CSV reports.
- Release process in lockstep with BOMHort: `release.yml` refuses a `vX.Y.0` tag unless the pinned BOMHort in `e2e.yml` is `X.Y`; E2E pinned to BOMHort v0.8.0-rc.3.

### Changed

- `Vulnerabilities`, `ProjectVulnerabilities` take a `*VulnerabilityOptions` (pass `nil` for the old behaviour); `ListVulnerabilities` takes `*VulnerabilityListOptions` instead of `*ListOptions`.
- `FindSBOM` no longer falls back to a full listing walk after a `?search=` miss (BOMHort's search already matches every candidate).
- `Retry-After` is capped at 30 s; a `RateLimiter` waiter cancelled via context gives its slot back.
- Typed errors (`APIError`, `IsNotFound`, `IsBadRequest`, `IsUnauthorized`, `IsForbidden`, `IsRateLimited`, `IsUnavailable`, `StatusCode`).
- Iterator-based pagination (`ListX` / `X` / `AllX`, `Paginate`, `Collect`).
- Options: API key, service token, HTTP client, user agent, rate limit / shared `RateLimiter`, 429 retries, strict decoding.
- `bomhorttest`: in-memory fake gateway modelling BOMHort ≥ 0.7 auth, pagination, uploads, duplicate detection, VEX scoping/product-wide/latest-wins matching and source `PATCH`, with canned responses for all other routes.
- Integration tests against a real BOMHort with strict decoding, docker-compose E2E harness, CI (lint, staticcheck, race tests on min/latest Go, coverage gate, govulncheck), weekly API-drift job against BOMHort `main`, release workflow.
- Documentation: API coverage, VEX contract, compatibility, testing, migration from VEXViper, releasing, contributing, AGENTS.md.

[Unreleased]: https://github.com/seebom-labs/bomhort-go/commits/main
