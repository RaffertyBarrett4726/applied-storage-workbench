# Tracing US/EU Browser Avatar Upload Failures Before Object Storage

Short answer: debug a browser avatar upload as two separate gates: CORS preflight first, then the signed storage write. In a US/EU customer-support system that retains signed documents until an explicit deletion deadline, the safest default is to keep the browser on a same-origin application endpoint unless the team can own, test, and observe the storage CORS policy for every production origin.

That choice is about failure ownership, not a theoretical preference for direct upload. A direct request can spare the application a byte-heavy hop, but it moves part of the upload SLO into browser policy, origin configuration, and regional storage behavior. A proxy costs API capacity and gives the platform team one place to validate files, record retention metadata, and enforce deletion. The right design is the one whose failure can be assigned to an owner at 02:00.

Preflight is a control-plane request.

## What evidence distinguishes a preflight denial from a failed write?

The browser is not asking the storage service whether a signed URL is valid when it sends an `OPTIONS` preflight. It is asking whether JavaScript from a particular origin may make a cross-origin request with a particular method and set of headers. For a non-simple cross-origin `PUT`, the preflight describes the requested origin, method, and headers; the response must authorize that combination before the browser releases the actual write.

The signature is a different gate. It authorizes a bounded operation, often including the object key, method, and expiration. It does not grant a web origin permission to call that operation, and it does not configure CORS. A correctly generated URL can therefore coexist with a browser error in which no object was written at all. That is why checking only the presign response creates a misleading green dashboard.

Start with the browser's Network panel, not the storage object's final URL. Record the `Origin`, `Access-Control-Request-Method`, and `Access-Control-Request-Headers` on the preflight, then compare them with the response's `Access-Control-Allow-Origin`, `Access-Control-Allow-Methods`, and `Access-Control-Allow-Headers`. A missing `OPTIONS` response, a production origin omitted from the allowlist, or a header added by a frontend HTTP wrapper is enough to stop the flow.

The US and EU detail matters because an allowlist is part of the deployed configuration, not an environment variable that can be assumed to match everywhere. Test the exact HTTPS origins used by the US and EU applications, including staging separately. Do not use `*` as a shortcut when credentials or private document access are involved; the origin and credential rules must match the browser request and the data's access policy.

## How do browser avatar uploads expose CORS and presigned PUT mistakes?

I use three checkpoints and give each one a distinct owner. This avoids turning every browser message into “storage is down,” which is a poor incident hypothesis and an even poorer capacity plan.

1. The application issues a short-lived authorization for the intended key, content type, and tenant. The API owns authentication, key naming, and retention metadata.
2. The browser completes preflight and sends the `PUT`. The web platform owns allowed origins, methods, and headers; the storage platform owns the bucket policy and request observability.
3. A server-side reader or metadata check confirms the object, its tenant association, and its deletion deadline. The data lifecycle owner owns retention and deletion verification.

For a failed request, capture a request correlation ID in the API response and log the object key without logging the signed URL. Then classify the result as preflight denied, actual write denied, transport failure, or post-write verification failure. A 403 from the write is not evidence of a CORS defect; a browser CORS message is not evidence that the signature was rejected. Those are different paths and deserve different alerts.

One failed preflight is enough to stop the write.

Here is a small Go model for the decision boundary. It intentionally does not call a commercial endpoint: the useful invariant is that a browser-direct path is eligible only when the exact origin and request shape are controlled. The production implementation should obtain these values from reviewed configuration and persist the resulting operation record alongside the customer-support document.

```go
package main

import "fmt"

type UploadPlan struct {
	Origin        string
	Method        string
	RequestedHead []string
	CorsOrigins   map[string]bool
	CorsMethods   map[string]bool
	CorsHeaders   map[string]bool
}

func directUploadAllowed(p UploadPlan) error {
	if !p.CorsOrigins[p.Origin] {
		return fmt.Errorf("origin is not allowed: %s", p.Origin)
	}
	if !p.CorsMethods[p.Method] {
		return fmt.Errorf("method is not allowed: %s", p.Method)
	}
	for _, header := range p.RequestedHead {
		if !p.CorsHeaders[header] {
			return fmt.Errorf("requested header is not allowed: %s", header)
		}
	}
	return nil
}

func main() {
	plan := UploadPlan{
		Origin:        "https://support.example",
		Method:        "PUT",
		RequestedHead: []string{"content-type"},
		CorsOrigins:   map[string]bool{"https://support.example": true},
		CorsMethods:   map[string]bool{"PUT": true},
		CorsHeaders:   map[string]bool{"content-type": true},
	}
	if err := directUploadAllowed(plan); err != nil {
		panic(err)
	}
	fmt.Println("preflight contract is satisfied")
}
```

This check is not a replacement for a real browser test. It is a compact way to make the contract testable before deployment. The test matrix should include both regional origins, a disallowed origin, the exact headers emitted by the frontend, an expired authorization, a retry after a lost response, and deletion verification at the deadline. I would keep the matrix in CI and run a small canary against each region after a CORS change.

## Where should retention and deletion state live for signed documents?

For signed customer documents, the upload path should create a server-owned record before bytes are accepted: tenant, opaque object key, permitted content type, maximum size, retention deadline, and deletion state. An avatar can use the same admission boundary even if it has a shorter lifecycle. The browser must never be allowed to choose a path that could escape the tenant prefix, and the signed operation should expire independently of the document's retention period.

That record is the part I don't leave implicit. Consider a support agent replacing an avatar while a signed attachment is being retained for an open case: the two objects may share an upload component, but they cannot share an unnamed lifecycle. The object key needs a tenant boundary; the record needs an owner; the deletion worker needs an idempotent state transition; and the verification job needs to distinguish “delete requested” from “delete confirmed.” If the browser times out after the storage service accepts bytes, the API must be able to reconcile the operation rather than create a second object on retry. If a customer changes regions, the record must still say which policy governs the document and which origin was allowed to upload it. This is why I treat CORS as one input to the retention workflow, not as the workflow itself. It’s a small distinction in a diagram and a large distinction during an audit.

The two common boundaries have different operational shapes:

| Boundary | What the team controls | Useful fit | Not suitable when |
|---|---|---|---|
| Same-origin proxy | Authentication, validation, capacity, storage credentials, deletion record | Small avatars, private documents, and teams that need one observable policy | Upload volume would consume the API's latency or throughput budget |
| Browser-direct signed write | Presigning, bucket CORS, browser test matrix, storage policy, post-write verification | Large payloads where the storage control plane is owned by the team | The team cannot change CORS, guarantee regional parity, or explain a failed preflight |

The catch is that direct upload does not remove operational work; it relocates it. A proxy needs headroom for accepted bytes plus retries, and direct upload needs headroom in the storage and browser control plane. Measure peak bytes per second, request size, retry amplification, and time to verified persistence separately from API latency. There is no universal payload threshold at which one boundary wins.

For deletion, do not treat an object-store lifecycle rule as the complete proof. The application record should say when deletion is due, a worker should issue the deletion operation, and a later verification pass should make the state explicit. If the system cannot demonstrate that signed documents disappear by the stated deadline, it has a retention-control problem even when avatar uploads look healthy. OWASP's upload guidance also supports allowlisted types, generated names, size limits, and storage outside a directly served webroot where applicable.

## When is a proxy the right rollback for direct upload?

Choose the proxy when the application team cannot own the bucket's CORS policy, when the browser request needs many changing headers, or when a private document must pass server-side inspection before any storage write is considered accepted. This is also the calmer choice for a beginner team: one same-origin endpoint is easier to exercise with an integration test, and its rejection reasons can be tied to the application's normal SLOs.

Rollback should be boring.

Choose direct upload only when the organization can review origin changes, test US and EU browsers, observe the write path, and reconcile an accepted upload with its retention record. A failed preflight should be a deployment signal, not a customer-discovered mystery. If a region's web origin cannot be included in the same tested policy, keep that region behind the proxy until the ownership boundary is real.

I would not claim that either design is universally faster. Your mileage may vary with browser network conditions, payload size, and the distance between users and storage. The decision record should name the SLO, the owner of each gate, the rollback path, and the evidence that deletion completed; a diagram without those four details is not an operating model.

## References

- [MDN: Cache-Control response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)
- [OWASP File Upload Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html)
- [MDN: CORS guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
