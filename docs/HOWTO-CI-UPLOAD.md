# How to: upload SBOM and VEX from CI

Push the SBOM your pipeline just built to BOMHort, wait until it is ingested
and scanned, then push the matching OpenVEX document scoped to that SBOM.

## Prerequisites

- BOMHort >= 0.7.0 with `AUTH_ENABLED=true` and writable storage (uploads
  answer 403 or 503 otherwise).
- An API key, stored as a CI secret (`BOMHORT_API_KEY` below).
- Go in the CI image, and the SBOM (`*.spdx.json` or `*.cdx.json`) plus,
  optionally, an OpenVEX file (`*.openvex.json`) as build artifacts.

## 1. Write the uploader

BOMHort ingests asynchronously and has no job-status endpoint: the upload
returns `pending` with a job id, and the SBOM shows up in `GET /api/v1/sboms`
once the worker is done. Give every run a unique filename so you can find
exactly this upload.

`tools/bomhort-upload/main.go`:

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"os"
	"time"

	bomhort "github.com/seebom-labs/bomhort-go"
)

func main() {
	sbomPath := flag.String("sbom", "", "SBOM file (*.spdx.json or *.cdx.json)")
	vexPath := flag.String("vex", "", "optional OpenVEX file (*.openvex.json)")
	name := flag.String("name", "", "unique filename for this run, e.g. app-<commit>.spdx.json")
	project := flag.String("project", "", "BOMHort project")
	repo := flag.String("repo", "", "source repository URL (https, no credentials)")
	ref := flag.String("ref", "", "git ref or commit")
	wait := flag.Duration("wait", 5*time.Minute, "how long to wait for ingestion and scan")
	flag.Parse()

	c := bomhort.New(os.Getenv("BOMHORT_URL"),
		bomhort.WithAPIKey(os.Getenv("BOMHORT_API_KEY")),
		bomhort.WithRateLimit(90, 10*time.Second),
	)
	ctx, cancel := context.WithTimeout(context.Background(), *wait)
	defer cancel()

	doc, err := os.ReadFile(*sbomPath)
	if err != nil {
		log.Fatal(err)
	}
	res, err := c.UploadSBOM(ctx, *name, doc, &bomhort.UploadOptions{
		Project: *project, SourceRepo: *repo, SourceRef: *ref,
	})
	if err != nil {
		log.Fatalf("upload SBOM: %v", err)
	}
	fmt.Printf("sbom upload: %s (job %s)\n", res.Status, res.JobID)

	// Wait until the SBOM is listed (ingested) and has findings (scanned).
	// An SBOM without any vulnerable package keeps vuln_count 0, so stop
	// waiting for the scan at the deadline instead of failing.
	var sbom bomhort.SBOM
	for {
		sbom, err = c.FindSBOM(ctx, *name)
		if err == nil && sbom.VulnCount > 0 {
			break
		}
		if ctx.Err() != nil {
			if err != nil {
				log.Fatalf("SBOM %s not ingested within %s: %v", *name, *wait, err)
			}
			fmt.Println("ingested, no findings yet (clean or scan still running)")
			break
		}
		time.Sleep(5 * time.Second)
	}
	fmt.Printf("sbom %s: %d packages, %d findings\n", sbom.ID, sbom.PackageCount, sbom.VulnCount)

	if *vexPath == "" {
		return
	}
	vex, err := os.ReadFile(*vexPath)
	if err != nil {
		log.Fatal(err)
	}
	// Scoping to sbom.ID makes every statement apply to this SBOM only.
	res, err = c.UploadVEX(ctx, *name+".openvex.json", vex, sbom.ID)
	if err != nil {
		log.Fatalf("upload VEX: %v", err)
	}
	fmt.Printf("vex upload: %s (job %s)\n", res.Status, res.JobID)
}
```

A `duplicate` status means BOMHort already has identical content (same
SHA-256); the SBOM from that earlier upload is the one `FindSBOM` returns if
the filename matched, otherwise search by document name instead.

## 2. Call it from the pipeline

GitHub Actions, after the step that produces `sbom.spdx.json` and
`vex.openvex.json`:

```yaml
- uses: actions/setup-go@v5
  with:
    go-version: stable
- name: Upload SBOM and VEX to BOMHort
  env:
    BOMHORT_URL: https://bomhort.example.com
    BOMHORT_API_KEY: ${{ secrets.BOMHORT_API_KEY }}
  run: |
    go run ./tools/bomhort-upload \
      -sbom sbom.spdx.json \
      -vex vex.openvex.json \
      -name "${{ github.event.repository.name }}-${{ github.sha }}.spdx.json" \
      -project "${{ github.event.repository.name }}" \
      -repo "${{ github.server_url }}/${{ github.repository }}" \
      -ref "${{ github.sha }}"
```

GitLab CI is the same command with `$CI_PROJECT_NAME`, `$CI_COMMIT_SHA`,
`$CI_PROJECT_URL` and a masked `BOMHORT_API_KEY` variable.

## 3. Verify

- The job log prints `sbom <uuid>: N packages, M findings` and, with VEX,
  `vex upload: pending`.
- In BOMHort the SBOM appears under the project with `source_repo` and
  `source_ref` set; after a few seconds the VEX'd findings show
  `vex_status: not_affected` (or whatever the statement says) with
  `vex_scope: sbom`:

```sh
curl -s -H "X-API-Key: $BOMHORT_API_KEY" \
  "$BOMHORT_URL/api/v1/sboms/<uuid>/vulnerabilities" | jq '.[] | {vuln_id, purl, vex_status}'
```

## When it doesn't work

| Symptom | Cause | Fix |
|---------|-------|-----|
| `HTTP 401` | Missing or wrong key | Check the secret; the header is `X-API-Key`. |
| `HTTP 403` on upload | Gateway runs with `AUTH_ENABLED=false` | Enable auth on the gateway; uploads are refused without it. |
| `HTTP 503` on upload | `SBOM_DIR` not writable | Mount writable storage for the api-gateway. |
| `HTTP 400 … X-Source-Repo` | Repo URL has credentials or is not http(s) | Pass the plain `https://host/org/repo` URL. |
| `not ingested within 5m` | Worker slow or failing | Check `ingestion-watcher` / `parsing-worker` logs; raise `-wait`. |
| Findings never get `vex_status` | Statement does not match | `vulnerability.name` and the subcomponent `purl` must equal the finding's `vuln_id` and `purl` byte for byte; see [docs/VEX.md](VEX.md). |
| `HTTP 429` persists | Several pipelines share one gateway | Share the limit: lower `WithRateLimit` or stagger the jobs. |

Reference: [README](../README.md), [docs/VEX.md](VEX.md),
[docs/API-COVERAGE.md](API-COVERAGE.md).
