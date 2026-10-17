# Compatibility & versioning

## Versions follow BOMHort

bomhort-go is released in **lockstep with BOMHort**: `bomhort-go vX.Y.*` is the client for `BOMHort X.Y.*`.

- **Major.minor** is BOMHort's. `v0.8.0` ships when BOMHort 0.8.0 is final and covers its complete public API; a BOMHort minor without API changes still gets a (trivial) bomhort-go minor so the mapping stays obvious.
- **Patch** is ours: client fixes, `bomhorttest` improvements, docs. `v0.8.3` still targets BOMHort 0.8.x.
- **Pre-releases** (`v0.8.0-rc.N`) track BOMHort release candidates when a consumer needs them early.
- A newer client works against the previous BOMHort minor for everything that minor has (new routes answer `404` → `IsNotFound`, new fields are empty, new query parameters are ignored). An older client works against a newer BOMHort because decoding is lenient by default.

| bomhort-go | Go | BOMHort | Tested in CI against |
|---|---|---|---|
| `v0.8.x` | 1.25+ (`go.mod`) and latest stable | 0.8.x | **v0.8.0** (pinned, every PR) · **`main`** (weekly) |

The pinned BOMHort ref lives in `.github/workflows/e2e.yml` (`BOMHORT_PINNED_REF`). Bump it — and this table — when a new BOMHort release (or RC) ships; see [RELEASING.md](RELEASING.md).

## Running against BOMHort 0.7

Routes and fields BOMHort added in 0.8.0 are marked `>= 0.8.0` in the Go doc comments and in [API-COVERAGE.md](API-COVERAGE.md). On a 0.7 gateway:

| Feature | On BOMHort 0.7.x |
|---|---|
| `Project`, `ListProjectSBOMs`, `ProjectVulnerabilities`, `ListProjectPackages`, `LicenseSources`, `VulnerabilityReportCSV` | `404` → `IsNotFound(err)` |
| `ListProjectGroups` (`?group_by=parent`), `LicenseViolationSummaries` (`?summary=true`) | parameter ignored; the server returns the plain rows, which decode into mostly-empty structs (and fail with `WithStrictDecoding`) |
| `VulnerabilityOptions.Scope` (`?scope=`), `SBOMListOptions.Project` (`?project=`), `UploadOptions.Parent` (`?parent=`) | ignored: everything is returned / no parent set |
| `Vulnerability.CVEID/Aliases/DependencyScope/DependencyDepth/AffectedSBOMs`, `DependencyNode.LicenseSource/DependencyScope/DependencyDepth`, `AffectedProject.DependencyScope/DependencyDepth`, `VEXStatement.CVEID/Aliases`, `SearchVulnerability.CVEID/Aliases`, `SBOMLicense.PackageScopes`, `LicenseViolation.UnapprovedCount/PackageScopes`, `Project.Parent/ParentSource`, `Tag.IsProject`, `UploadResult.Parent` | empty |
| `Dependencies` of an unknown SBOM id | `200 []` instead of `404` |

The integration suite skips `>= 0.8.0` routes when the server answers `404` ([TESTING.md](TESTING.md)).

## Go API stability

- **0.x:** the exported Go API may change between minors; every change is listed in [CHANGELOG.md](../CHANGELOG.md) with migration notes. We avoid gratuitous breaks: BOMHort additions are mirrored as new fields and methods, and a removed or renamed BOMHort field stays as a deprecated Go field for at least one minor.
- **1.x (once BOMHort's API is frozen):** no breaking changes to exported identifiers within a major.
- The client never decodes strictly by default, so a newer BOMHort that adds fields does not break an older client.
- `bomhorttest` is part of the public API and versioned with it; its behaviour tracks the BOMHort release pinned in CI.

## Go version policy

The `go` directive in `go.mod` is the minimum supported version; CI tests it and the latest stable release. The minimum is raised only when a needed language or stdlib feature requires it (currently range-over-func iterators and `iter.Seq2`, Go 1.23+, and the toolchain in use).
