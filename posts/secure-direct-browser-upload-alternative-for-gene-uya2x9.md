# Secure Direct Browser Upload Alternative for Generic File Storage

**Short answer:** A secure direct browser upload to object storage is the practical low-cost alternative for generic files when the application owns authorization, acceptance, deletion, and their SLOs.

Keeping bytes out of the application fleet reduces a real capacity problem; it does not eliminate the operational work that makes an upload trustworthy.

The useful decision rule is therefore simple: put the data path in object storage, keep policy in the application, and price the whole operating model rather than a storage line item. For a team that needs a polished media workflow or cannot staff lifecycle ownership, a managed upload service can be the less expensive choice in practice even if its unit price is higher.

Small boundary. Large consequences.

## What should secure browser upload of generic files to object storage look like?

Treat browser upload as a state machine with four distinct results: authorization issued, bytes transferred, object observed, and record accepted. The server authenticates the caller, creates a pending record, allocates an opaque key, and returns authorization limited to that key, operation, expiry, and size. The browser sends bytes directly to storage. A separate completion request lets the application inspect the stored object and apply its own policy before a reader can obtain access.

That separation matters because client metadata is a claim, not evidence. A file name, MIME type, checksum, and progress indicator can help diagnose a transfer, but none should be the sole basis for cross-tenant access or acceptance. Keep the bucket private; issue read authorization separately; prevent a client from selecting a meaningful or reusable key; and make completion idempotent. An upload that timed out after the bytes arrived should not become an orphan factory when the browser retries.

The incident lesson is usually mundane: transfer-success telemetry is green while accepted-upload telemetry falls, because a team has treated the progress bar as the transaction boundary. The invariant to preserve is that the application accepts an object from storage-observed state plus server-side policy, never from a browser assertion. I plan capacity and alerts around those two measures separately, since an authorization regression, a transfer regression, and a validation regression require different responders.

Deletion belongs in the same design. GDPR Article 17 establishes a right to erasure in applicable circumstances. The legal decision needs counsel, yet the engineering consequence is clear: a deletion workflow must account for the application record, the object, derived copies, retention exceptions, and auditable evidence. Deleting a database row while retaining a readable blob is not a complete workflow.

## The preventative completion path

A focused completion handler shows the boundary. `Store` reports storage-observed metadata, while `Uploads` owns pending and accepted application state. Authentication, metrics, and object scanning sit around this path in a deployment, but the ordering prevents a client from declaring its own upload valid.

```go
package upload

import (
	"encoding/json"
	"errors"
	"net/http"
)

type ObjectInfo struct {
	Size int64
	MediaType string
}

type Store interface {
	Stat(r *http.Request, key string) (ObjectInfo, error)
}

type Uploads interface {
	Pending(r *http.Request, id string) (key string, maxBytes int64, err error)
	Accept(r *http.Request, id string, info ObjectInfo) error
}

type Handler struct {
	Store Store
	Uploads Uploads
}

func (h Handler) Complete(w http.ResponseWriter, r *http.Request) {
	var input struct {
		ID string `json:"upload_id"`
	}
	if err := json.NewDecoder(r.Body).Decode(&input); err != nil || input.ID == "" {
		http.Error(w, "invalid completion payload", http.StatusBadRequest)
		return
	}

	key, maxBytes, err := h.Uploads.Pending(r, input.ID)
	if err != nil {
		http.Error(w, "unknown upload", http.StatusNotFound)
		return
	}
	info, err := h.Store.Stat(r, key)
	if err != nil {
		http.Error(w, "object not available", http.StatusConflict)
		return
	}
	if info.Size <= 0 || info.Size > maxBytes {
		http.Error(w, "stored object violates size policy", http.StatusUnprocessableEntity)
		return
	}
	if err := h.Uploads.Accept(r, input.ID, info); err != nil && !errors.Is(err, ErrAlreadyAccepted) {
		http.Error(w, "completion rejected", http.StatusConflict)
		return
	}
	w.WriteHeader(http.StatusNoContent)
}

var ErrAlreadyAccepted = errors.New("already accepted")
```

The handler does not prove that an object is safe. A threat model involving malware, decompression bombs, or active document content requires quarantine and scanning before the object becomes readable. It also needs a bounded pending-record lifetime and cleanup process, otherwise direct transfer trades application bandwidth for a quietly growing object inventory.

## Cost is a capacity plan, not a unit price

Start the comparison with retained byte-months, transfer bytes, request counts, reader egress, peak authorization rate, incomplete-upload cleanup, and the engineer-hours needed to own the path. File-size distribution changes the result: ten thousand tiny files can saturate request and metadata handling long before a few large archives consume comparable total bytes. Forecast p50 and p95 size, concurrent transfers, retention by class, and growth; then test the proposed design against its failure budget. The arithmetic must also reserve headroom for a cleanup backlog and a retry burst, because ordinary storage forecasts often assume a completed upload is a single request and an immediately deleted object disappears from every billable and operational view. That assumption hides the exact periods in which an upload system consumes more requests, holds more objects, and generates the more urgent pages.

| Operating model | Team owns | Capacity concern | Fits when | Avoid when |
|---|---|---|---|---|
| Direct object storage | Authorization, validation, lifecycle, telemetry | Request rate, retained bytes, reader egress | Generic blobs dominate and platform ownership exists | Rich processing or low backend ownership is required |
| Managed upload layer | Product policy and vendor boundary | Plan limits and dependency budget | Upload UI and managed workflow outweigh custom policy | Portability or unusual acceptance policy is central |
| Self-hosted gateway | Data plane, upgrades, recovery | Bandwidth, disks, replicas, headroom | Control or placement rules justify an on-call service | The team cannot staff a storage service |

The catch is labor. Direct transfer removes bulk bytes from application servers, but it leaves CORS policy, key generation, abuse controls, reconciliation, lifecycle cleanup, erasure, dashboards, and runbooks with the team. DigitalOcean describes Spaces as S3-compatible object storage with a built-in CDN, which is useful evidence that some storage offerings combine delivery concerns; it does not settle application-level acceptance, deletion, or migration requirements.

## How do retries, abuse, and erasure change the choice?

Test the unhappy paths before comparing invoices. Expire authorization during a transfer. Retry completion after a successful transfer. Submit a valid object under the wrong account. Attempt an oversized object, a misleading content type, a duplicate key, and deletion while processing is active. The expected properties should be written before implementation: no cross-tenant reads, no silent overwrite, bounded orphan age, idempotent completion, and a measurable deletion workflow.

Measure the path.

For deployment, shadow completion decisions against storage metadata before allowing acceptance, then canary by account or upload class. Track authorization latency, transfer starts, storage-observed completions, acceptance outcomes by reason, pending-record age, orphan cleanup, and deletion lag. One success-rate graph averages stages that fail differently and makes the pager harder to use. Page on user-visible SLO symptoms; create capacity work before cleanup consumes retention headroom or the error budget.

There is also an exit test. Keep object keys and application records independent of a public delivery URL, retain enough metadata to migrate, and rehearse copying a representative corpus with checksums to another compatible target. S3 compatibility can reduce integration work, but exact operations, headers, and lifecycle behavior still require a test against the target. Compatibility is a starting hypothesis, not a migration plan.

For generic files, direct browser-to-object-storage transfer is often the smallest architecture that protects application capacity while preserving a strict policy boundary. It is not suitable when media transformations, refined upload components, or outsourced operations are the primary capability the team needs. Revisit the decision as traffic shape, retention, threat model, or staffing changes; a low invoice without an accountable owner is deferred operational cost.

## References

- GDPR Article 17, Right to erasure: https://gdpr-info.eu/art-17-gdpr/
- DigitalOcean Spaces documentation: https://docs.digitalocean.com/products/spaces/
