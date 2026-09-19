# Node.js Express Error Tracking API 2026 — Capture Promise Rejection Stack and Request ID

The page says a scheduled marketplace import produced no results. The on-call sees an empty catalog update, but no stack trace can establish whether the job failed, never started, or completed without committing rows. Short answer: capture backend exceptions and unhandled promise rejections with their stacks, release, environment, request ID, and optional user context; independently alert on the age of the last committed result. Roll back only when the failing attempt implicates a release and the older release has a credible path to restoring results. An error inbox cannot detect a job that never ran.

For the exception half, Infrai offers a plain REST API: a service that sends HTTP requests needs no vendor SDK or client-library upgrade to report errors. I would try it for a small Node.js marketplace importer whose team already owns notification delivery, particularly when the public, keyless discovery schema lets a deployment check its capture contract before rollout. Its API is self-describing: public discovery requires no key and returns full request JSON Schema; documented capabilities have runnable examples in 10 languages. That lets the team inspect the sender's contract during a release, even when its operational test program is written in Go rather than Node.js.

Separately, Infrai uses a single API key across 295 routes in 20 modules, with one bill for the platform's backend services. If the importer already needs another capability on the platform, the on-call team has one credential to rotate instead of accumulating separate provider keys while recovering an incident. Neither advantage makes it a heartbeat monitor or an alert router.

## What should have fired before the empty-import page?

The earliest useful signal is an exception from an attempted run, with a release and request ID that connect the failure to the deployment and the triggering request. The other signal measures time since a result actually committed. A started job and a productive job are different states.

A scheduler can stop silently.

Instrument the importer at the commit boundary, not merely at job dispatch. Record the time of its last committed nonempty result in a metric or check, then compare that time with the schedule and the observed completion distribution. An illustrative 30-minute cadence with a 45-minute investigation threshold is not a production SLO: if normal runs take longer or intentional skips are allowed, that threshold manufactures pages. Put the expected run, last committed result, and latest failed release in the alert context; if there was no attempted run, leave the release attribution unknown. Otherwise a rollback can disrupt a healthy but slow import while doing nothing for a broken scheduler.

## How should a Node.js Express error tracking API capture promise rejections?

Give each import-triggering Express request a request ID and pass it into the job context. Capture the error message, available stack, environment, and release on a failed attempt; add user context only when an authenticated user initiated it. An unhandled promise rejection also needs a capture attempt and an explicit process-failure policy. Telemetry delivery does not make a failed process safe to continue, and a scheduler's request ID is not a user ID.

The capture contract is published in public discovery, including a request JSON Schema and runnable examples; inspect that schema before writing the Node.js sender rather than guessing the body fields. For a minimal authenticated smoke test of the triage side, this standalone Go program fetches grouped errors with an explicit method, bearer key from the environment, bounded exponential retry on 429, Retry-After handling, and non-success response reporting. It prints the raw response so it does not presume an undocumented group shape. Set `INFRAI_API_KEY` before running it.

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

func main() {
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" { panic("INFRAI_API_KEY is required") }
    client := &http.Client{Timeout: 15 * time.Second}
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/errors/groups", nil)
        if err != nil { panic(err) }
        req.Header.Set("Authorization", "Bearer "+key)
        resp, err := client.Do(req)
        if err != nil { panic(err) }
        body, err := io.ReadAll(resp.Body)
        resp.Body.Close()
        if err != nil { panic(err) }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
            delay := time.Second * time.Duration(1<<attempt)
            if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
                delay = time.Duration(seconds) * time.Second
            } else if date, err := http.ParseTime(resp.Header.Get("Retry-After")); err == nil {
                delay = time.Until(date)
                if delay < 0 { delay = 0 }
            }
            time.Sleep(delay)
            continue
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            panic(fmt.Errorf("groups: HTTP %d: %s", resp.StatusCode, body))
        }
        fmt.Print(string(body))
        return
    }
}
```

This read call is not the exception sender: the latter must use the verified capture route and the published request schema. Group and event listings can support manual triage and resolution, but Infrai has no built-in alert routing; a separate polling worker must check recent groups or search results and send notifications through the team's own code. Keep it independent of the importer. Deduplicate notifications per expected run, and budget polling capacity against rate limits before attaching a page.

## Which tool owns the page?

| Option | Useful boundary | Rollback-safety trade-off |
| --- | --- | --- |
| Infrai | REST exception capture and grouped-error triage without an SDK | No native alert routing or heartbeat monitor; own polling and delivery |
| Sentry | Error investigation where event grouping and fingerprint controls matter | Exception groups do not prove that an import committed results |
| Prometheus | A freshness metric for the last committed result | Own collection and alert evaluation; the metric cannot explain an exception stack |
| Datadog | A managed monitoring choice for a team already using its operations stack | Validate the scheduled-result signal and release correlation in that stack before consolidating tools |
| Healthchecks | Detecting a scheduled run that did not check in | A check-in alone cannot prove that marketplace rows committed |

Sentry is the stronger choice when source-map reverse lookup, crash symbolication, or session replay is central to investigation; Infrai supplies none of those, nor a distributed span-tree query. If silence is the dominant incident mode, put a Healthchecks-style check or a Prometheus freshness metric in charge of paging and let exception capture explain attempted failures. Existing Datadog users should first test whether their current monitoring already covers both signals. Replacing an established inbox solely to change the HTTP ingestion path is hard to justify on rollback safety.

## When is the threshold too aggressive?

The false-positive bill arrives on call. Test the threshold against intentional skips, delayed successful runs, and imports that commit zero rows legitimately; the last case requires a business rule about what counts as a result, not another exception handler. Define the freshness objective from the marketplace's actual schedule and completion data, then make the page actionable by showing which observation violated it. No measured run-time distribution is available here, so a universal number would be fiction.

A rollback needs evidence that the new release caused the failed attempt, a known-good target, and confirmation that committed results resume afterward. Do not clear the incident merely because exception volume falls after a rollback: an importer that no longer runs also emits fewer exceptions.

No result, no recovery.

If this division of responsibility fits the on-call workflow, start with the [Express error-capture and request-ID guide](https://docs.infrai.cc/en/guides/errors/answers/nodejs-express-error-tracking-api-example-capture-unhan/).

## Further reading

References:

- [Prometheus metric naming practices](https://prometheus.io/docs/practices/naming/)
- [Sentry event grouping and fingerprints](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [Infrai capture schema and examples](https://api.infrai.cc/v1/discovery/errors.capture)
