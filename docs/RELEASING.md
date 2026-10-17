# Releasing

bomhort-go versions follow BOMHort ([COMPATIBILITY.md](COMPATIBILITY.md)): `vX.Y.0` is cut for BOMHort `X.Y.0`, `vX.Y.Z` (Z > 0) are client-only fixes. Releases are tags on `main`; `.github/workflows/release.yml` does the rest.

## When BOMHort publishes a release (or RC)

1. Diff BOMHort's `backend/cmd/api-gateway/main.go` and `backend/pkg/dto/api.go` against the previous ref (`git diff v0.7.1..v0.8.0 -- backend/cmd/api-gateway backend/pkg/dto`). For every new route, field, query parameter or status-code change: method/field + doc comment with the version (`>= 0.8.0`), `endpoints_test.go` row, `bomhorttest` behaviour if observable, integration test, [API-COVERAGE.md](API-COVERAGE.md) row, CHANGELOG entry.
2. Pin it: set `BOMHORT_PINNED_REF` in `.github/workflows/e2e.yml` to the release's **commit SHA** (`git ls-remote https://github.com/seebom-labs/BOMHort.git 'refs/tags/vX.Y.Z^{}'`) and update the table in [COMPATIBILITY.md](COMPATIBILITY.md).
3. Run `make check` and `make e2e` (`BOMHORT_SRC` at that ref). The E2E run must be green with strict decoding; an `unknown field` error means step 1 missed something.
4. Merge; the PR's E2E job repeats step 3.

For an RC this is all. For the final release continue below.

## Cutting a release

1. `main` is green (CI and E2E against the pinned BOMHort).
2. One commit `git commit -s -m "Release vX.Y.Z"`:
   - `const Version = "X.Y.Z"` in `bomhort.go`;
   - move the `## [Unreleased]` entries of `CHANGELOG.md` under `## [X.Y.Z] - YYYY-MM-DD` and state the BOMHort version it targets in the first line of the section.
3. Tag and push:
   ```sh
   git tag -s vX.Y.Z -m vX.Y.Z   # or -a if you don't sign tags
   git push origin main vX.Y.Z
   ```
4. `release.yml` re-runs `make check`, verifies that the tag equals `bomhort.Version`, that `CHANGELOG.md` has a section for it and — for `vX.Y.0` — that the pinned BOMHort ref in `e2e.yml` is annotated with the same `X.Y`; then it creates the GitHub release with the changelog section as notes (tags containing `-` become pre-releases) and asks `proxy.golang.org` to fetch the version.
5. Afterwards set `Version` to the next `-dev` version (`X.Y.Z+1-dev`, or `X.Y+1.0-dev` once work on the next BOMHort minor starts) and add an empty `## [Unreleased]` section.

## Module path

The module path is `github.com/seebom-labs/bomhort-go`. The proxy step only runs in that repository; in forks the release is created but `go get` will not resolve it.
