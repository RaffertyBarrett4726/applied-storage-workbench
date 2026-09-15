# Property Receipts: Node.js SaaS File Export Delivery with Signed URLs and Object Storage

**Short answer:** for a property-management SaaS that processes receipts and keeps the original file for audit, use private object storage with short-lived signed URLs when files must outlive app instances; use app-server files only when their lifetime, backup, and recovery are deliberately bounded.

Retention must be decided before delivery is optimized. Store the original behind an explicit receipt record, keep it private, and let a signed URL deliver bytes only after the application has checked authorization and deletion state. The cheap-versus-simple argument comes later; a file that cannot be explained or recovered is expensive regardless of where it sits.

The incident pattern is ordinary. A monthly property report is generated, a leasing coordinator downloads one plumbing receipt, and a cleanup task later sees a file that appears old enough to remove. The dangerous question is not “can the browser download it?” It is “what does this byte sequence belong to, who may still retrieve it, and what evidence permits deletion?”

I model the receipt row as the control plane: property scope, receipt identity, object key, checksum, retention deadline, deletion state, and any audit hold. The URL is a disposable capability produced from that row. That distinction keeps an exported report from quietly becoming the system of record.

Three words matter: retain, release, delete.

## The rollout starts with a deletion ledger, not a download link

The application should commit metadata only after the original write has completed and its checksum has been computed. A `ready` state means the system can identify the bytes, their owner, and the deadline; it does not merely mean a worker finished pushing something somewhere. A report can have a separate generated artifact, but the original receipt needs its own identity and retention decision.

The download handler then checks the requester against the property scope, verifies that the row is still releasable, and asks the storage boundary for a time-limited signed URL. Signed URLs are bearer credentials. They reduce the need for the API server to relay a large response, but they do not replace authorization, audit logging, or deletion policy. A copied link should expire; a revoked or deleted receipt should not be downloadable just because an earlier link has not reached its nominal expiry.

This is the state transition I want reviewers to be able to draw on a whiteboard:

`queued -> ready -> expired -> deleted`

An audit hold can prevent the final transition without changing who is authorized to read the receipt. That separation is useful in operations: a legal or audit decision pauses deletion, while an application permission decision governs release.

Here is the small piece worth testing first. It makes key construction, checksum capture, and the retention timestamp independent of a storage SDK, so the same tests can run against a local adapter and a managed object store.

```go
package receipts

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"regexp"
	"time"
)

var safeID = regexp.MustCompile(`^[A-Za-z0-9_-]+$`)

type OriginalReceipt struct {
	Key          string
	Checksum     string
	RetentionEnd time.Time
}

func PrepareOriginal(propertyID, receiptID string, data []byte, retentionEnd time.Time) (OriginalReceipt, error) {
	if !safeID.MatchString(propertyID) || !safeID.MatchString(receiptID) {
		return OriginalReceipt{}, fmt.Errorf("invalid receipt identity")
	}
	if len(data) == 0 {
		return OriginalReceipt{}, fmt.Errorf("receipt is empty")
	}

	digest := sha256.Sum256(data)
	return OriginalReceipt{
		Key:          fmt.Sprintf("receipts/%s/%s/original", propertyID, receiptID),
		Checksum:     hex.EncodeToString(digest[:]),
		RetentionEnd: retentionEnd.UTC(),
	}, nil
}
```

The code does not upload anything, by design. The write adapter should accept the prepared key and bytes, verify completion, and return enough metadata for the database transaction. If the transaction fails after a successful write, reconciliation must find the unowned object and apply a documented policy; silently deleting it is unsafe when an audit hold could exist.

## What receipt evidence survives a retention review?

The storage choice changes the failure boundary, not the retention obligation. An application instance can make a local file look convenient while deployment replacement, replicas, disk pressure, backup scope, and recovery testing remain someone else’s problem. Object storage can separate file lifetime from compute, but it adds lifecycle configuration, credential ownership, transfer verification, and inventory work. A shared filesystem sits between those choices and brings its own mount and throughput dependencies.

| Boundary | Useful property | Retention risk to own | Fit for original receipts |
|---|---|---|---|
| App-server disk | Few moving parts for a small workload | Replacement, replicas, backups, and disk loss | Only when lifetime and recovery are deliberately bounded |
| Shared filesystem | Familiar file semantics across processes | Mount availability, permissions, capacity, and failover | An existing, operated boundary with tested recovery |
| Private object storage | File lifetime is decoupled from compute | Lifecycle rules, credentials, reconciliation, and transfer checks | Strong default when originals outlive application instances |

The delivery mechanism should follow that boundary. For a private object, the app authorizes first and issues a short-lived link second. For a file on an application volume, the app may stream it directly, but that does not make the volume durable. For a shared filesystem, direct reads avoid a relay but still leave the team responsible for access controls and capacity.

Capacity planning needs four separate measures: rendering time, readiness delay, download latency, and deletion backlog. The first two consume worker CPU and temporary space; the third consumes network and storage reads; the fourth consumes reconciliation and cleanup capacity. One “download SLO” hides the interesting failure, which may be a receipt that remains retained for months without an owner or a cleanup queue that cannot prove what it removed.

I would set alerts around missing ownership, checksum mismatch, objects past their retention deadline, pending deletion holds, and repeated generation retries. Your mileage may vary on thresholds because property count, receipt size, and retention duration drive the shape of the workload. I’m not sure any universal cost ranking survives those inputs, so I would measure them before promising a savings number.

## How should Node.js SaaS reports integrate signed URLs for file export downloads?

Generation should be asynchronous when reports can be large: create the receipt and report intent, write the original, verify the write, commit `ready`, and expose a download action. The action rechecks scope and state before issuing the signed URL. Cleanup marks an eligible row, confirms that no hold is active, deletes through the adapter, and records the result idempotently. A retry after a successful deletion should be a harmless no-op, not a second destructive operation with a different interpretation.

The operational contract needs more than a happy-path test. Exercise an unauthorized property, a receipt with an expired deadline, a retry after the object write but before the database commit, a duplicate receipt identity, a checksum mismatch, and cleanup racing with an audit hold. Keep the object key, checksum, retention decision, authorization scope, and timestamps as durable evidence. Logs are useful for diagnosis; they are a poor substitute for the record that explains why deletion was allowed.

If an existing system starts on instance disk, migration is a controlled change in ownership. Inventory the database rows and files, copy to unique keys, verify bytes and metadata, record old and new locations, then switch reads only for verified records. Reconcile missing references before deleting the old location. I would run that in bounded batches with a pause point, because rollback means restoring a known read path, not guessing which copy happens to be newer.

That process also makes the SLO honest. “All receipts migrated” is not a useful measurement unless it includes verified bytes, mapped metadata, and a reconciliation result. A 99th-percentile download number cannot compensate for one original receipt whose retention deadline has no enforceable owner.

## What must the evaluation prove before deletion?

Private object storage is unsuitable when every byte must be transformed through the application, when the workload is permanent anonymous publishing, or when the team cannot operate credentials, lifecycle rules, and reconciliation. A signed URL alone is also insufficient for a legally immutable archive; that contract needs independently enforced holds and tamper-evident controls.

Stick with app-server storage for disposable exports when the instance lifetime, backup, and recovery test are explicit. Use a shared filesystem when its operator already owns capacity and failover. Choose an archive-oriented system when immutability is the requirement. The right answer is the boundary that can keep the retention promise after a deployment, retry, and quiet period.

## References

- [AWS S3 object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Google Cloud Storage documentation](https://cloud.google.com/storage/docs)
