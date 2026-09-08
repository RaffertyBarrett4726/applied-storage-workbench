# Tenant-Isolated Gaming Exports: Node.js Presigned Download URL Signature Mismatch 403s

Tenant isolation changes the answer. **Short answer: keep every training artifact under an immutable, tenant-scoped object key; mint the download URL only after the export is complete; and treat a 403 as a signed-request investigation before changing bucket visibility.** A file export link should carry a bounded authorization, while the application remains the authority for tenant, object key, retention class, and export state.

This matters in a game studio where a training run may produce checkpoints, replay shards, evaluation reports, and a human-readable export. A link that works for one tenant but names another tenant's key is a security incident. A link that expires in the worker queue is an SLO miss. Those are different failures, even when the browser displays the same status.

## Tenant isolation starts with the export retention ledger

Start with a record, not a URL. The export record should bind a tenant identifier to an immutable object key, the artifact's content type and size, its retention class, and the state of the export job. The key can be structured like `tenants/tenant-42/runs/run-918/checkpoints/model-003.bin`; the exact naming scheme is yours, but the tenant component must come from an authenticated authorization decision rather than from a filename supplied by the browser.

The download handler checks that binding, confirms the artifact is ready, and only then signs a GET for the exact raw key. It should never accept a complete object URL from the client and should not rebuild a key by decoding and re-encoding pieces from a query string. Store the canonical key once. Encode it once at the request boundary.

Three words are useful here: identity, bytes, time. Identity is the tenant and object key. Bytes are the exact signing input, including path and query parameters. Time is the validity interval and the clock used to create it. A 403 signature mismatch means at least one of those dimensions differs from what the storage service verifies; it does not mean that public access is the correct remedy.

## How do Node.js object storage downloads recover from a presigned URL 403 signature mismatch?

The usual suspects are narrower than they look. The signer may have used a different bucket or key than the metadata check. A space, plus sign, percent sign, or slash may have been normalized by a framework, proxy, or client. A cached link may have expired while a large training export waited behind other work. The signer and verifier may also disagree about time because their clocks are skewed.

Key encoding deserves a test case of its own. Suppose the database stores `reports/season 3/replay+summary.json`. Signing an already escaped value and escaping it again changes the request. Treating `+` as a space changes it too. The reliable comparison is not two pretty URLs in a browser: it is the raw bucket, raw key, issuance timestamp, expiry timestamp, and untouched URL captured at the signing boundary. Redact the signature itself. For a concrete triage pass, read the canonical key from the export row, log a safe digest or an escaped diagnostic representation of that raw value, and compare it with the value passed into the signing library before any router parameter is decoded. Then compare the path in the returned URL without copying it into a new URL builder, because a second builder may normalize a percent-encoded segment or reorder query parameters. If the metadata check uses `tenant-42` and `reports/season 3/replay+summary.json` but the signer uses a reconstructed path from a display filename, stop there: the object lookup and the signed request are about different bytes. If both use the same raw values, move to timestamps; do not keep changing encoding until one probe happens to pass.

Clock skew is a capacity-planning problem when the artifact is produced asynchronously. Let the validity window cover the worst credible export queue delay, download pickup delay, and transfer time, with a small measured clock margin. Do not make every URL live for days just to hide a queue problem; that widens the exposure window and makes revocation harder. I’m not sure what margin your fleet needs until its clock offset is measured, so record the relevant timestamps and check the time-synchronization status of signing hosts.

The browser adds another branch. CORS controls whether browser JavaScript may read a cross-origin response; it does not make a bad signature valid. A command-line GET that succeeds while browser JavaScript reports a CORS failure is a browser-policy issue. An actual 403 carrying a signature complaint remains a signed-request issue.

## A focused runbook for Node.js file export links

The service can be written in Node.js, Go, or another language; the boundary is what matters. Read the tenant-scoped export record, verify readiness and retention eligibility, perform a metadata check using the raw bucket and key, mint a fresh URL, and send that URL unchanged to the downloader. The downloader performs an ordinary GET without adding the application's bearer authorization header.

Keep the diagnostic small.

No URL surgery.

| Signal | Inspect first | Action |
| --- | --- | --- |
| Fresh URL is 403 | Raw tenant, bucket, and key at both boundaries | Stop issuance and compare signing inputs |
| Old URL is 403, fresh URL works | Issuance and expiry timestamps | Reissue only for a ready, retained export |
| CLI works, browser JavaScript fails | Browser CORS response and console | Fix browser policy separately from signing |
| Link expires in the queue | Queue age, pickup delay, and clock offset | Issue later or revise the measured validity budget |

For a gaming export, the verification probe should exercise the same path users cross, but it should not load an entire checkpoint into memory. A streaming client can preserve the artifact while recording status, headers needed for diagnosis, and the number of bytes received. The following Go example illustrates the separation between an authenticated metadata probe and an unauthenticated signed-URL GET without inventing a provider-specific signing request.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

func get(ctx context.Context, client *http.Client, target, bearer string) (int, int64, error) {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, target, nil)
	if err != nil {
		return 0, 0, err
	}
	if bearer != "" {
		req.Header.Set("Authorization", "Bearer "+bearer)
	}

	response, err := client.Do(req)
	if err != nil {
		return 0, 0, err
	}
	defer response.Body.Close()

	bytes, err := io.Copy(io.Discard, response.Body)
	if err != nil {
		return response.StatusCode, bytes, err
	}
	if response.StatusCode < 200 || response.StatusCode >= 300 {
		return response.StatusCode, bytes, fmt.Errorf("GET %s returned %s", target, response.Status)
	}
	return response.StatusCode, bytes, nil
}

func main() {
	metadataURL := os.Getenv("METADATA_URL")
	signedURL := os.Getenv("PRESIGNED_URL")
	token := os.Getenv("API_TOKEN")
	if metadataURL == "" || signedURL == "" || token == "" {
		panic("set METADATA_URL, PRESIGNED_URL, and API_TOKEN")
	}

	client := &http.Client{Timeout: 30 * time.Second}
	ctx, cancel := context.WithTimeout(context.Background(), 90*time.Second)
	defer cancel()

	if _, _, err := get(ctx, client, metadataURL, token); err != nil {
		panic(fmt.Errorf("metadata probe failed: %w", err))
	}
	status, bytes, err := get(ctx, client, signedURL, "")
	if err != nil {
		panic(fmt.Errorf("signed download failed after %d bytes: %w", bytes, err))
	}
	fmt.Printf("verified signed download: %s, %d bytes\n", http.StatusText(status), bytes)
}
```

The environment variables are deliberately abstract. The metadata endpoint must be the endpoint your storage contract actually defines, and the presign request must follow that contract's published schema. A generic code sample should not pretend that two storage APIs have interchangeable paths or signing fields. In production, replace `io.Discard` with a file or object writer and compare the received byte count with the export record.

## When should the export be reissued, rolled back, or rejected?

The decision should be deterministic. If metadata says the tenant-key binding is missing, reject the request and create an audit event. If metadata succeeds but an old URL returns 403, issue a fresh URL only when the record is still ready and within its retention policy. If a fresh URL also fails, stop issuing links for that export class, preserve the raw diagnostic fields, and page the owner of the signing path. Do not turn the bucket public as an incident response.

For retention, use immutable keys per export rather than overwriting `latest.bin`. A reproducible policy needs to answer which bytes belonged to which run and when those bytes may be deleted. If an overwrite has already removed the prior bytes and no retained copy exists, rollback cannot recreate them; that is a data-recovery boundary, not a URL problem.

Before reopening the path, test a key containing a space, `+`, `%`, and nested slash segments, plus one deliberately expired URL. Confirm that a request signed for tenant A cannot retrieve tenant B's key. Run a small batch within the observed concurrency envelope and watch the successful-download SLI, queue age, 403 rate, and expiration age. Three checks. Then scale.

The catch is that this design is not suitable when you need permanent public links, provider-native object locking, or a storage platform's specific replication and lifecycle controls. In that case, stick with the direct provider integration that supplies those controls and accept the extra credential, client, and on-call ownership. A managed abstraction is a reasonable fit when a plain HTTP boundary and centralized policy matter more than those provider-specific controls, but neither choice removes the obligation to verify tenant identity, exact key bytes, and time.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://aws.amazon.com/s3/pricing/
