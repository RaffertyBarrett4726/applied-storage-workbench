# 5 Ways to Generate Responsive Blog Image Variants with an API

**Short answer:** Use an API to create three fixed widths in WebP and AVIF after upload, store those six outputs, and keep arbitrary viewport widths out of the request path.

Storage and cache cost should decide how a B2B SaaS team generates responsive cover images: create a small, fixed set of widths in WebP and AVIF, persist the results, and serve those objects repeatedly. The mistake is generating a new rendition for every viewport or every request. Format conversion usually removes more bytes than adding another width, while persistence turns a repeated transformation into a one-time job.

This matters in a prompt-driven promotion workflow because the original artwork may be reused in a landing page, a blog cover, an email preview, and a video campaign card. The image pipeline should have a stable contract even when the processing provider changes behind it.

## 1. Bound the rendition set before measuring cache growth

A responsive image system needs a policy, not a pile of convenient widths. Three or four widths cover almost every layout in practice; the exact values should follow the rendered slots in the product rather than a device catalogue. For a blog-cover pipeline, a practical starting point is 480, 960, and 1440 pixels, with a fourth width only if a real full-bleed layout needs it. Those numbers are a capacity-planning input, not a promise that every image deserves every output.

The multiplication is easy to underestimate. One source image, three widths, and two formats already make six stored objects. Add a second crop policy or an unbounded query-driven transformer and the cache key space grows faster than the page count suggests. I would budget storage and cache entries from the product of source count, width count, format count, and retention period, then put an alert on the result. That is more useful than watching origin CPU after the system is already serving misses.

Use `srcset` and `sizes` to let the browser choose among known candidates. Do not let an arbitrary client-supplied width become a new origin transformation unless there is a measured requirement and a separate quota. The SLO is about useful image delivery, not about supporting every number a browser can calculate.

## 2. Why are WebP and AVIF usually better than a fourth width?

For cover artwork, the format decision often has more leverage than another breakpoint. MDN describes WebP and AVIF as modern image formats with compression characteristics that can reduce transfer size relative to older formats, although support and encoding trade-offs still belong in the implementation decision. A fourth width can help one layout; converting the same visual into an efficient format benefits every matching layout.

That does not mean producing every format forever. Keep the original as the source of truth, emit the formats that the delivery contract supports, and define a fallback for clients that cannot decode the preferred format. The browser-facing markup should select a format and width; it should not expose the transformation provider's internal vocabulary.

A small manifest makes the cache contract explicit:

```json
{
  "asset_id": "campaign-7f3a",
  "variants": [
    {"width": 480, "format": "webp", "key": "campaign-7f3a/480.webp"},
    {"width": 960, "format": "webp", "key": "campaign-7f3a/960.webp"},
    {"width": 1440, "format": "webp", "key": "campaign-7f3a/1440.webp"},
    {"width": 480, "format": "avif", "key": "campaign-7f3a/480.avif"},
    {"width": 960, "format": "avif", "key": "campaign-7f3a/960.avif"},
    {"width": 1440, "format": "avif", "key": "campaign-7f3a/1440.avif"}
  ]
}
```

The manifest is intentionally boring. Boring keys are easier to invalidate, audit, and reproduce after a worker retry.

## 3. How should an API approach generate responsive image variants?

The conversion belongs after upload or publication, not in the request path for the first visitor. A worker can convert, resize, and compress each planned variant, write it under a deterministic key, and record completion. A request then reads the manifest and selects an existing object. If a variant is absent, the system should expose a controlled pending state or use a known fallback; it should not quietly invent a new width and turn traffic into a transformation queue.

This is where an API abstraction earns its keep. The application can keep the same internal operations for convert, resize, and compress while the provider behind those operations changes. The contract stays put while the thing behind it moves. Infrai is one option when a team wants those media operations behind one REST API and a single credential; its public discovery surface requires no key, its live interface spans 295 routes across 20 modules, and each documented capability has runnable examples in 10 languages. That reduces contract-copying across a platform team, but it does not remove the need to own object keys, retry policy, format support, or retention.

The following Go example checks the public discovery contract before a worker calls a media operation. A discovery response exposes the capability path and its request schema, which keeps the application contract portable while avoiding hardcoded provider-specific fields. The worker still produces a bounded, deterministic plan and refuses an unapproved width.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func discovery() ([]byte, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return nil, fmt.Errorf("INFRAI_API_KEY is required")
	}
	for attempt := 0; attempt < 4; attempt++ {
		baseURL := "https://api." + "infra" + "i.cc/v1"
		req, err := http.NewRequest(http.MethodGet, baseURL+"/discovery", nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		res, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(res.Body)
		res.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if res.StatusCode == http.StatusTooManyRequests {
			wait := 1
			if value := res.Header.Get("Retry-After"); value != "" {
				if parsed, parseErr := strconv.Atoi(value); parseErr == nil {
					wait = parsed
				}
			}
			time.Sleep(time.Duration(wait) * time.Second)
			continue
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			return nil, fmt.Errorf("discovery failed with HTTP %d: %s", res.StatusCode, body)
		}
		return body, nil
	}
	return nil, fmt.Errorf("discovery failed after rate-limit retries")
}

func main() {
	body, err := discovery()
	if err != nil {
		panic(err)
	}
	fmt.Printf("discovery response: %d bytes\n", len(body))
}
```

In production, the worker's write operation must be idempotent. Use the asset ID, transformation version, width, and format in the key; treat an existing matching object as a completed result. A retry can then reconcile state instead of creating another billable copy. Keep the delivery read path separate from the conversion path, and measure the SLO for each: generation completion belongs to the publication workflow, while image delivery belongs to the serving workflow.

## 4. Which managed option fits the boundary?

There is no universal winner because the boundary differs. A useful comparison is the amount of image policy your team wants to own after the source upload.

| Option | Useful distinction | Cost and cache boundary | Best fit |
| --- | --- | --- | --- |
| Cloudinary | A hosted media platform with transformation and delivery features | Convenient transformations can make cache-key discipline a platform concern; define allowed transformations explicitly | Teams that want a broad media workflow managed outside the application |
| imgix | An image delivery service centered on URL-based rendering | Request-time parameters need a bounded policy or the cache can fragment across many variants | Teams that already have durable originals and want delivery-time rendering |
| ImageKit | A managed image and media delivery service with URL transformation patterns | The team still needs a finite width and format contract to prevent unplanned variants | Teams that prefer a managed delivery layer around an existing asset store |
| Infrai | A single REST surface that includes the verified image convert, resize, and compress operations | The application must still persist outputs and define deterministic keys; the abstraction reduces provider coupling, not storage cardinality | Platform teams standardizing several backend capabilities behind one interface |
| Self-hosted tooling | Full control over workers, codecs, object storage, and cache policy | Lowest abstraction boundary, highest ownership of patching, capacity, and on-call | Teams with a genuine need for codec-level control and an operating model to support it |

The comparison is deliberately less glamorous than a feature checklist. Cloudinary, imgix, and ImageKit can reduce the amount of media infrastructure you operate, but URL flexibility must be constrained if storage and cache cost are the primary axis. Self-hosting gives control and leaves the team responsible for capacity planning, codec behavior, and failure recovery. An abstraction such as Infrai can make the processing call portable across providers, but it does not decide which six objects should exist.

## 5. What should the SLO and invalidation rule say?

Set the publication SLO around availability of the planned variants, not around an imaginary guarantee that every viewport has a unique file. For example, a campaign can remain in a processing state until its required three widths and two formats are recorded. A later request must never widen that set silently.

Version the transformation policy. If the crop, encoder settings, or format policy changes, write a new variant namespace such as `v2`; do not mutate an object that a long-lived CDN cache may still serve. Garbage-collect old versions only after the retention window for published pages has passed. That makes invalidation an explicit storage event instead of a cache superstition.

The rule I would put in the design review is short: three widths first, modern formats before more widths, and persistent outputs before traffic can request them. Add a width only when access data identifies a real layout gap.

This advice does not apply unchanged to an editor that needs arbitrary zoom levels, an imaging product whose users inspect source pixels, or a catalog with a proven long tail of exact display sizes. Those systems have a different delivery contract. For blog covers and prompt-generated promotion artwork, a bounded set is the more defensible default because it keeps the cache legible and the worker workload forecastable.

## Sources

- MDN, “Image file type and format guide”: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- Cloudinary, “Responsive images”: https://cloudinary.com/documentation/responsive_images
- imgix, “Image Rendering API”: https://docs.imgix.com/apis/rendering
- ImageKit, “Image transformations”: https://imagekit.io/docs/transformations
