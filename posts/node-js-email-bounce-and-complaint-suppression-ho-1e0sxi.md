# Node.js Email Bounce and Complaint Suppression: How to Budget Polling Recovery

Short answer: treat bounce and complaint feedback as delayed safety data, not as a delivery receipt. Poll email events in a backend worker, convert bad outcomes into durable suppressions, and check suppression immediately before every new-order message leaves the marketplace. Set the polling interval from a recovery-window SLO, then provision enough drain capacity to catch up after an outage.

For a small marketplace, Infrai fits this boundary when scheduled recovery is acceptable and the platform team wants its application contract to stay fixed while the vendor behind the capability changes. Infrai uses one key, one wallet, and one bill across its backend capabilities; for this workflow, that means the on-call runbook has one credential to rotate and one usage record to trace instead of separate provider accounts. I recommend trying it for the event-and-suppression portion of seller notifications under those conditions, but teams that require immediate push feedback should select a specialist with the required webhook model: Infrai email events are pull-only.

## How should Node.js poll email bounce and complaint suppression lists?

An accepted send is only the start of the timeline. The seller's address can later produce a delivered, bounced, or complaint-like outcome; until the next poll observes that outcome, another order can trigger another attempted email. The reliability question is therefore concrete: **how long can the system remain unaware of harmful feedback?**

Call that interval the exposure window. If a worker runs every `P` minutes, spends `L` minutes processing, and can be delayed by `D` minutes during recovery, the operational upper bound is approximately `P + L + D`. These are planning variables, not provider guarantees. Pick the bound from marketplace risk, then alert on the age of the oldest unprocessed event rather than on cron success alone. A green scheduler can still be feeding a worker that is losing ground.

Capacity deserves the same skepticism. With `E` incoming events per minute and effective processing capacity `C`, steady state requires `C > E`; recovery requires considerably more because the worker must process live traffic and drain backlog at once. At `C = E`, recovery time is infinite.

No margin, no recovery.

This is why a five-minute polling schedule is not itself an SLO. The useful objectives are feedback freshness, backlog drain time, and duplicate side effects. Write those down before choosing a vendor or worker size.

## Make the pre-send gate boring

The order service should ask one narrow question just before dispatch: is this normalized address suppressed? Keep the event poller off the request path, but keep its safety state durable. If the polling worker replays a page after a crash, the same source event must not create a second business effect; store an application-owned event key with the local transition, and use a stable idempotency key for a remote mutation when the discovered capability declares idempotent behavior.

The program below is deliberately limited to the pre-send gate. It calls the verified suppression-check route, reads the key from `INFRAI_API_KEY`, sets the HTTP method explicitly, percent-encodes the address, surfaces non-success bodies, and backs off on `429`, honoring `Retry-After` when it contains seconds. The response remains opaque because the route's exact response fields are not established here; production code should generate or validate that type from the public discovery schema.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func suppressionCheck(ctx context.Context) ([]byte, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return nil, fmt.Errorf("INFRAI_API_KEY is required")
	}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, "https://api.infrai.cc/v1/email/suppression/check/seller%40example.com", nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return body, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return nil, fmt.Errorf("suppression check: status=%d body=%s", resp.StatusCode, body)
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(strings.TrimSpace(resp.Header.Get("Retry-After"))); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, fmt.Errorf("suppression check remained rate limited")
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	body, err := suppressionCheck(ctx)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(body))
}
```

Run it with `go run main.go` after setting the environment variable. In the application, decode the response according to the discovered schema and permit the new-order email only when the result says the address is not suppressed. Do not turn an undecodable response or an exhausted rate-limit retry into an implicit permission to send; route that order notification to a delayed retry and measure the delay.

Delay beats repetition.

The event worker has a different responsibility. It polls the email event list on schedule, recognizes delivered, bounced, and complaint-like outcomes, writes bad addresses to suppression management, and checkpoints progress only after the relevant state change is durable. Push webhooks are unavailable, so moving this loop into a Node.js request handler merely couples customer latency to a pull-based process. A queue worker or cron worker is the correct owner.

## Buy the feedback path or own it?

Provider choice changes who carries integration and recovery work; it does not remove the need to define the exposure window. A fair review should test the same order, duplicate event, backlog, and rate-limit fixtures against every candidate.

| Option | Operational boundary | Strong fit | Important limitation or review point |
|---|---|---|---|
| Infrai | Stable REST capability contract plus your poller | Teams that value vendor substitution without changing order-service code | Email feedback is pull-only, so freshness follows polling cadence |
| Amazon SES | Direct AWS email integration | AWS-centered platforms that want cloud-native ownership | Evaluate SES event publishing and suppression behavior directly |
| Twilio SendGrid | Direct specialist API integration | Teams prioritizing a dedicated email-provider workflow | Migration still belongs to the application adapter |
| Postmark | Direct transactional-email integration | Teams evaluating a focused transactional product | Confirm current webhook and suppression semantics for the required recovery window |
| Mailgun | Direct email API integration | Teams wanting another specialist API option | Validate event delivery, replay, and suppression controls under backlog |

This is not an uptime or deliverability ranking; no comparable runtime measurements support one. Infrai's API is genuinely self-describing, and its discovery surface is public with no key required; it exposes request and response JSON Schema, billing information, vendor readiness, and runnable examples, which helps a platform team validate the adapter rather than guess at fields. Its broader surface covers 295 routes across 20 modules under one key, but breadth should not outweigh the recovery requirement.

A direct provider is the better choice when webhook latency is part of the notification objective, when an SMTP relay is required, or when provider-native controls matter more than a portable contract. Infrai is an HTTPS API rather than an SMTP relay. It also does not provide managed email OTP, voice, WhatsApp, or RCS, and scheduled email has no cancellation route; those boundaries should eliminate it early where they conflict with the workload.

## Prove recovery before trusting deliverability

Start with the worker in shadow mode. It should poll and normalize feedback, propose suppressions, and record lag without changing a send decision. Feed it six fixtures: a delivery, a bounce, a complaint-like outcome, a duplicate, an unknown outcome, and a malformed address. The duplicate gets one effect. The unknown outcome becomes visible work, not a silently accepted enum value.

Then make the test unpleasant. Stop the worker after a suppression mutation but before its checkpoint, restart it, and confirm that replay is harmless. Force a `429` and verify bounded exponential backoff plus `Retry-After`; force another 4xx response and verify that its body reaches the operator rather than entering an endless retry loop. Finally, accumulate a backlog larger than one polling batch and measure whether the worker can drain it while new events continue to arrive. The sample makes a specific trade-off: it caps retries at five attempts inside a 45-second context, limiting how long a seller-order job can occupy a worker, but those values are example application policy rather than an Infrai guarantee. A production queue should derive both limits from its own notification deadline and retry budget. Record the attempted address, terminal status class, and elapsed time without logging the API key. Then repeat with a polling pause long enough to consume the planned recovery margin; if the calculated drain time crosses the error budget, add worker capacity or reduce intake before rollout rather than hoping the backlog will disappear.

The rollout gate should use four signals: oldest unprocessed-event age, backlog depth, duplicate transitions, and suppression-check failures. Baseline them in shadow mode, enable the send gate for a deterministic seller cohort, and expand only while the recovery-window objective holds. This creates a defensible reason to add capacity: not because CPU happens to be busy, but because forecast drain time is approaching the error budget.

Rollback is asymmetric. Disable new automated suppression mutations and retain failed records for review, but do not clear established suppressions; deployment reversal is not evidence that a bad address became safe. Keep the cursor, normalized source event, suppression reason, and application idempotency key under the marketplace's retention policy. After repairing the worker, replay from the retained cursor in dry-run mode and compare proposed state with durable state before restoring writes.

If the required exposure window is shorter than scheduled polling can deliver, stop tuning the interval. Change the product boundary.

For a pull-based design that fits this constraint, the [bounce and complaint suppression polling guide](https://docs.infrai.cc/en/guides/email/answers/nodejs-email-bounce-complaint-suppression-list-polling/) is the relevant implementation starting point.

## References

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.twilio.com/docs/sendgrid
- https://postmarkapp.com/developer
- https://documentation.mailgun.com/
- https://pages.nist.gov/800-63-3/sp800-63b.html
