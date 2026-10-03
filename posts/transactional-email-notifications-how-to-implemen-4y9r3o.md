# Transactional Email Notifications: How to Implement SMS Fallback with Status Polling

A customer-support compliance notice is not done when an API accepts it. The operational constraint is later proof: the team must reconstruct what was sent, which delivery state was observed, and why SMS was allowed to follow email. **TL;DR:** use email as the primary channel, poll delivery events on a schedule, and release a short SMS fallback only through a persisted policy decision. Choose a webhook-capable provider instead when the evidence SLO cannot tolerate polling delay.

This design has a hard ownership boundary. The provider transports messages and reports state; the application owns escalation timing, retries, evidence retention, and cross-channel policy. A consolidated service can reduce credential and invoice sprawl to one key and one bill, but consolidation does not transfer those controls to the vendor.

Consider a bounded audit exercise for support notice `NOTICE-1042`. I start at the end: can an investigator reproduce the SMS decision from stored inputs without searching application logs? If the only records are “email accepted” and “SMS sent,” the answer is no. The missing artifact is the decision itself, including the rule version and the observations available at that moment. That gap, rather than the mechanics of sending mail, determines the implementation.

## How should Node.js event notifications poll transactional email and SMS status?

Build one evidence package per notice, then make the delivery worker populate it. The package should identify the notice revision and recipient, retain the provider message identifier, append every status observation with its observation time, and record the exact policy outcome that reserved the fallback. Hashing the raw response gives later reviewers a way to detect alteration; it does not make an unvalidated response authoritative.

Keep three assertions separate. Provider acceptance proves that a request crossed an API boundary. A later transport event describes delivery state. Neither proves that a person read or understood the notice. DMARC addresses domain-level email authentication and reporting, not human acknowledgement, while NIST's authenticator guidance is a reminder that a notification fallback should not quietly become an improvised authentication protocol.

The invariant is short: **a channel transition must be reproducible from immutable observations and a versioned rule.** A mutable `status` column fails that test because each poll destroys the state that preceded it.

This also changes message content. The email may carry the full compliance notice; the SMS fallback should remain a short alert or urgent pointer rather than copying sensitive case material onto a lock screen. Retention, access, and encryption rules belong to the organization's compliance policy, not to a debugging default.

## Step 1: Poll before making the escalation decision

The smallest useful component polls one verified event route and produces a tamper-evident observation for the decision worker. It does not send either channel. That separation prevents a replay during an audit, test, or worker restart from sending another message, while keeping provider response parsing behind an adapter whose schema can be reviewed independently.

The following Go program is runnable with `go run .` after saving it as `main.go` and setting `INFRAI_API_KEY` plus `INFRAI_BASE_URL`. It uses an explicit method, treats non-success bodies as errors, retries `429` with exponential backoff, honors an integer `Retry-After`, and writes the raw-response hash that belongs in the evidence package. It deliberately does not guess the event response fields; production code must validate the current response schema before converting any field into a normalized delivery state.

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func poll(ctx context.Context, client *http.Client, baseURL, key string) ([]byte, error) {
	endpoint := strings.TrimRight(baseURL, "/") + "/v1/email/event/list"
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
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
			return nil, fmt.Errorf("poll returned %d: %s", resp.StatusCode, strings.TrimSpace(string(body)))
		}

		delay := time.Second << attempt
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		case <-time.After(delay):
		}
	}
	return nil, fmt.Errorf("poll exhausted retries")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	baseURL := os.Getenv("INFRAI_BASE_URL")
	if key == "" || baseURL == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY and INFRAI_BASE_URL are required")
		os.Exit(2)
	}
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	body, err := poll(ctx, &http.Client{Timeout: 15 * time.Second}, baseURL, key)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	sum := sha256.Sum256(body)
	fmt.Printf("notice_id=%s observed_at=%s response_sha256=%s\n",
		"NOTICE-1042", time.Now().UTC().Format(time.RFC3339), hex.EncodeToString(sum[:]))
}
```

After schema validation, feed the normalized observation to a pure reducer that emits `wait`, `close-email`, or `reserve-sms`; it must not send directly. Persist `reserve-sms` under a unique key such as `(notice_id, rule_version, action)` before a separate outbox worker sends anything. The write request also needs a stable idempotency key, an explicit HTTP method, status checking, surfaced error bodies, and the same `429` discipline. Local uniqueness covers competing workers; remote idempotency covers an ambiguous retry after the request crossed the network. They solve different failures, and skipping either one leaves a specific duplicate-send window.

Do not normalize arbitrary strings into `delivered`. Validate provider responses against the current contract first, store the raw-response hash, and keep the adapter version beside the normalized observation. One surprising field should fail closed, not authorize a second channel.

## Step 2: Derive the poll budget from the SLO

Pull-only status changes the capacity equation. If 120,000 notices remain open and the scheduler checks each one every 120 seconds, steady-state demand is 1,000 polls per second before retries. That arithmetic is illustrative capacity planning, not a measured provider limit, but it exposes the variable that matters: open evidence windows, rather than daily send volume.

Set separate SLOs for provider acceptance, evidence freshness, and fallback timeliness. Then calculate the maximum poll interval from the narrowest remaining budget after queue delay and retry allowance. Index work by `next_poll_at`, lease due records, add jitter, and close expired evidence windows. Watch the oldest overdue poll. Averages hide the notice that breaches policy.

Short version: bound the open set.

No poller makes pull behave like push.

Polling also constrains recovery. Both email and SMS tracking are pull-only in this option, so a scheduler outage creates an evidence backlog and delays channel transitions. There is no webhook path to drain that gap immediately. If the business requires pushed state changes, polling is the wrong control regardless of how convenient the send API looks.

## Step 3: Choose the control surface, not the logo

The buy-versus-build question is who operates the evidence path. These products expose materially different boundaries; none removes the need for an application-owned compliance record.

| Option | Status mechanism | What the platform team still owns | Best fit |
|---|---|---|---|
| Amazon SES with Amazon SNS | SES event publishing into AWS destinations | IAM, event routing, state normalization, and the evidence store | Teams already operating AWS as their control plane |
| Twilio SendGrid with Twilio Messaging | SendGrid Event Webhook and Twilio status callbacks | Callback authentication, two product schemas, and cross-channel policy | A pushed transition is part of the SLO |
| Mailgun with Twilio Messaging | Mailgun webhooks or Events API plus Twilio callbacks | Multiple credentials, contracts, adapters, and failure domains | Channel specialization is worth the additional operations |
| Infrai | Scheduled polling for email and SMS state | Poll capacity, normalization, deadlines, and all fallback safeguards | Bounded evidence lag is acceptable and one key and one bill reduce operational sprawl |

Infrai is credible in the last row because one plain REST API covers the channels without another required SDK, and its self-describing discovery surface lets a reviewer inspect current schemas without a production key. The trade is substantial: neither namespace pushes webhook events, there is no SMTP relay for a legacy mailer, and the application must call the email API directly. Email has no managed OTP interface. Scheduled email cannot be canceled, although SMS has a cancellation operation; if revocation matters, hold email in an application queue until release instead of scheduling it remotely. Voice, WhatsApp, and RCS are outside the channel set, and a pending domestic email vendor cannot support a domestic-compliance conclusion.

The other rows are not interchangeable. Amazon SES is attractive when IAM and AWS event infrastructure are already staffed. SendGrid and Twilio provide pushed callbacks, at the cost of securing public callback ingestion and reconciling separate product models. Mailgun plus Twilio keeps channel vendors replaceable and specialized, while leaving more credentials and contracts on the on-call inventory. The explicit limitation is that Infrai is not suitable when the SLO requires webhook delivery, when a legacy SMTP integration must remain intact, when scheduled email must be remotely canceled, or when voice, WhatsApp, or RCS is required; choose the matching pushed or channel-specific product instead. I would reject any option whose status mechanism misses the evidence-freshness SLO before comparing developer ergonomics or billing.

## Step 4: Put policy ahead of the fallback worker

An SMS reservation must pass business-layer controls for recipient authority, suppression, country allowlists, per-recipient throttling, and country-based spend circuit breakers. Those safeguards are not implied by transport acceptance. Cost reporting cannot be aggregated by tag through this API, and SMS template inventory should be tracked by the application rather than assumed to be discoverable through a list operation.

Run the audit test again after implementation. Given only the evidence package, can a reviewer identify the notice revision, follow every observed state, reproduce the versioned decision, and distinguish provider acceptance from delivery? If yes, the worker is serving the compliance job. If not, adding another retry will not repair the design.

**Use email with SMS fallback only when the organization accepts application-owned orchestration and the poll interval fits its evidence SLO.** Prefer one channel when collecting a phone number is unjustified, a webhook-capable product when status must be pushed, and human review when delivery metadata would otherwise trigger a consequential customer action.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [NIST SP 800-63B: Authentication and Authenticator Management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Twilio Message status callbacks](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Mailgun webhooks](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)

## Sources

- https://datatracker.ietf.org/doc/html/rfc7489
- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html
- https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- https://www.twilio.com/docs/messaging/guides/track-outbound-message-status
- https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks
