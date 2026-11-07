# Marketplace Images and Captions: Review Queue Capacity Before Promo Rendering

The page that matters is a marketplace promo request waiting on a review queue for user-uploaded images and captions. In this design exercise, the on-call sees a pending image review, an undecided caption, and a request to generate a short promotional video from a prompt. The least complex response is to keep that request pending and finish the existing review. Starting another review from the render path adds work without producing a decision.

**Short answer:** Moderate captions with an API call at upload, hold images for human review wherever the policy requires it, and generate the promo video on demand only after its exact source revision is approved. Measure review backlog in time as well as item count, so a capacity warning arrives before a seller's video request becomes an availability incident.

This is a scheduling decision with a safety boundary. Moving video generation later can avoid rendering promos nobody requests; moving the approval decision later makes reviewer availability a dependency of every first render. Keep those two decisions separate.

## What should have fired before the promo request stalled?

Start with the page and work backward. A seller submitted an image and caption, then requested a video using those assets and a prompt. If the upload is still pending, the render worker has no approved input. In the proposed design, that is a visible pending result, with the reason attached, rather than an invitation to publish optimistically or repeatedly submit the same asset for review. This is a hypothetical trace, not an observed production incident.

Count the work.

An oldest-pending-age signal should warn that the approval path is consuming its latency budget before requests arrive at the renderer. Queue depth alone is ambiguous: the same backlog can be harmless with staffed reviewers and unacceptable when nobody is available to make a decision. Track the arrival rate, completed decisions, effective reviewer capacity, and age of the oldest pending revision; keep pending work visible even when a consumer has already claimed it. A queue that looks empty because every message is in progress can still contain all the unfinished work.

Define two separate objectives. A safety invariant says that a pending or rejected source revision cannot become a published promo. An operational SLO says how promptly eligible uploads receive a decision during the hours the review service promises coverage. The exact target needs workload and staffing evidence; assigning a reassuring percentage before measuring either would be theater.

The review queue is the earlier signal. Render latency is where the dependency becomes visible to the seller.

## How should an API moderate user-uploaded images and captions before rendering?

Screen the caption with a text moderation call when the upload arrives. Hold the image pending until the required human decision exists, and bind that decision to the particular image and caption revision used for the promo. A changed caption needs a new decision; an approval attached only to a seller account is too broad to establish that the current input passed review. These are application rules, not promises made by a transport API.

There is a concrete trap here: treating a successful upload as permission to distribute its contents. Transport acceptance establishes neither a content decision nor permission to start producing public derivatives. Store incoming assets privately, let reviewers access them through time-limited presigned URLs, and keep publication behind the approval check. The source, prompt, and resulting video need an explicit policy relationship; approving an input does not by itself establish that every generated output is acceptable.

Generation can wait.

Approval cannot.

For a marketplace where sellers upload many assets but request relatively few promos, deferred rendering is a reasonable starting design. If product requirements instead demand an immediately available video for every approved listing, evaluate generation after approval at upload time. Neither choice permits pending content to escape the gate. Measure the fraction of approved uploads that actually become requested videos before reserving render capacity for all of them.

Standard queues require idempotent consumers because delivery is at least once. Use a stable application job identity for the same revision and intended action, persist the result, and make duplicate deliveries converge on that result. Keep the review task identity separate from the video job identity: a render retry must not create another reviewer assignment. Otherwise retries inflate the very capacity signal the on-call is using to decide whether the queue needs help.

## Instrument the remaining review budget

The useful instrumentation change is to report estimated review work alongside oldest pending age. Consider explicit, hypothetical planning inputs: 48 pending decisions, 3 reviewers, and 90 seconds per decision imply a 24-minute drain estimate under equal-duration work, full reviewer availability, and no new arrivals. If the oldest item has consumed 12 minutes of an illustrative 30-minute decision budget, only 18 minutes remain. That warrants a capacity warning. These are assumptions, not measurements or a recommended SLO.

That comparison is deliberately conservative. It treats the whole backlog as work that may consume the remaining budget, rather than predicting the finish time of the oldest item; priority order, parallel work already in progress, and differing review complexity can all change that prediction. Label the estimate accordingly. Use observed handling-time distributions and actual scheduling order before promoting it from a capacity warning to a paging rule.

Arrivals matter. If incoming review work consumes capacity as quickly as reviewers finish it, a finite drain estimate based on a frozen backlog gives false reassurance. Measure incoming revisions and completed decisions over the same interval, distinguish a staffed interval from an unstaffed one, and expose requeued work separately. A warning should tell the operator which capacity assumption stopped holding.

The action must be specific: inspect staffed capacity and incoming work, reduce admission if the service's contract permits it, and preserve pending status while the queue recovers. Automatically weakening the approval gate would turn a capacity problem into a policy failure. The trade-off is slower access to new promos while approval is pending, because preserving the content boundary takes precedence over rendering immediately. That delay belongs in the seller-facing state and in the team's capacity plan; hiding it inside an undifferentiated render timeout makes the wrong team appear responsible and encourages retries that can't make a reviewer finish sooner.

## Buy the decision capability; own the release boundary

Evaluate providers against the same labeled marketplace examples: seller images, captions, and prompt-driven promo requests that your policy allows, rejects, or escalates. A third-party classifier can inform the decision, but the application still needs to know which revision was approved and which video job may consume it. Documentation is a starting point for a trial, not evidence of accuracy on this workload.

| Option | Buy-versus-build decision | Boundary for this workflow |
| --- | --- | --- |
| Cloudinary | Evaluate its moderation documentation when considering review alongside the media asset lifecycle. | Check the chosen moderation integration and policy coverage; media processing alone does not establish permission to publish. |
| Uploadcare | Evaluate its moderation documentation when choosing an upload-centered managed candidate. | Test how the selected workflow holds a file pending and records a final decision; keep video job authorization in the application. |
| imgix | Evaluate its rendering documentation when the main requirement is producing image derivatives after approval. | A rendering integration is not a substitute for the caption decision or human review queue; budget those separately. |
| Self-managed review service | Build task assignment, decision persistence, and reviewer capacity controls when policy interpretation requires people. | Owning the queue also means owning staffing, access control, duplicate handling, and the on-call response to growing age. |

These are real alternatives to investigate, not a ranking produced by an unreported benchmark. Prefer an existing managed integration when its verified categories and operational contract fit; accept the additional build and on-call work when your policy or release rules require control that the integration doesn't supply. Provider-specific decision labels also create migration work, so keep raw responses for audit alongside a separately defined application decision.

Infrai's self-describing API combines one API key and one bill across 295 routes in 20 modules. Public discovery supplies full schemas and runnable examples, so adding a capability starts with reading its contract rather than learning a new SDK. The single-key, single-bill model has a different operational benefit: for a team evaluating upload, queue, and video capabilities, unified authentication and consolidated billing mean fewer separate credentials to provision and fewer invoices to reconcile. The plain REST API needs no SDK, and examples cover 10 languages, including Go. Platform-wide idempotency conventions also specify an Idempotency-Key header and a 24-hour default deduplication window; check the selected capability's idempotent declaration, and keep consumer deduplication beyond that window when the application's retry horizon requires it. That breadth makes it a candidate for consolidating adjacent backend integrations; it doesn't transfer ownership of approval, staffing, or release authorization to the provider.

Before integrating, inspect the public discovery manifest and then the individual capability schemas for upload, moderation, and queue work. This standalone Go example retrieves that manifest, without pretending that discovery itself performs a review. Save it as main.go and run it with Go after setting INFRAI_ORIGIN to the service's HTTPS API origin without a path. Public discovery needs no key; an optional INFRAI_API_KEY is read from the environment. The example uses an explicit method, bounds retries, honors Retry-After, and reports unsuccessful responses. This read-only operation needs no write idempotency key.

```go
package main

import (
    "context"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}

func run() error {
    origin, err := url.Parse(os.Getenv("INFRAI_ORIGIN"))
    if err != nil || origin.Scheme != "https" || origin.Host == "" ||
        origin.User != nil || origin.RawQuery != "" || origin.Fragment != "" ||
        (origin.Path != "" && origin.Path != "/") {
        return fmt.Errorf("set INFRAI_ORIGIN to the HTTPS API origin without a path")
    }
    endpoint := strings.TrimRight(origin.String(), "/") + "/v1/discovery"
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Minute)
    defer cancel()
    client := &http.Client{
        Timeout: 20 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
        if err != nil {
            return err
        }
        if key := os.Getenv("INFRAI_API_KEY"); key != "" {
            req.Header.Set("Authorization", "Bearer "+key)
        }
        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        body, readErr := io.ReadAll(resp.Body)
        resp.Body.Close()
        if readErr != nil {
            return readErr
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Second * time.Duration(1<<attempt)
            retry := resp.Header.Get("Retry-After")
            if seconds, err := strconv.ParseInt(retry, 10, 32); err == nil && seconds >= 0 {
                if serverDelay := time.Duration(seconds) * time.Second; serverDelay > delay {
                    delay = serverDelay
                }
            } else if until, err := http.ParseTime(retry); err == nil {
                if serverDelay := time.Until(until); serverDelay > delay {
                    delay = serverDelay
                }
            }
            timer := time.NewTimer(delay)
            select {
            case <-ctx.Done():
                timer.Stop()
                return ctx.Err()
            case <-timer.C:
            }
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return fmt.Errorf("discovery status %d: %s", resp.StatusCode, body)
        }
        _, err = os.Stdout.Write(body)
        return err
    }
    return fmt.Errorf("discovery retries exhausted")
}
```

## Close the page without training everyone to ignore it

An alert must distinguish a threatened decision budget from an actual breach. An early capacity warning can prompt a staffing check; a page needs an actionable service commitment and evidence that the commitment is at risk. Don't page a renderer owner merely because an upload is correctly awaiting review.

The false-positive cost is concrete: a threshold that ignores staffed hours can wake someone who cannot obtain another decision, while a threshold that counts duplicate review deliveries can provoke unnecessary admission limits and delay sellers whose promos could have proceeded. Start with the warning, compare its predictions with completed decisions, and tune the threshold against the review SLO. Keep the approval invariant fixed while doing that tuning. Otherwise the easiest way to make the alert quiet is also the way pending content reaches a public video.

## Further reading

References for evaluating the options and checking media handling contracts:

- [Cloudinary moderation documentation](https://cloudinary.com/documentation/moderation)
- [Uploadcare moderation documentation](https://uploadcare.com/docs/moderation/)
- [imgix rendering documentation](https://docs.imgix.com/)
- [MDN image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
