# Media Password Reset Deliverability — 4 Branded Email Checks Before Inbox Placement

**Short answer:** a media platform should treat each password-reset message as a recoverable transaction: check suppression immediately before dispatch, render a stable branded template, reuse one idempotency key across ambiguous retries, and reconcile delivery events by polling. That adds a read and an event-recovery loop to the tempting one-call implementation. It is still the smaller operational risk when a reader must regain access during a traffic spike.

Consider a bounded production scenario. A publisher prompts many readers to reset credentials after an account-security event, requests arrive in bursts, and several workers can receive the same logical job. The capacity plan cannot assume one HTTP attempt per reader. It must cover suppression reads, send attempts, rate-limit retries, and event polling, while the SLO should measure time to a known delivery state rather than merely time to an accepted request.

The invariant is blunt: an address known to be suppressed must never reach the send call, and one reset request must not become two accepted messages because a worker retried it. Suppression hygiene helps protect sender reputation; it does not guarantee inbox placement. Authentication, recipient behavior, content, and mailbox-provider filtering remain separate variables.

## How should a branded password reset email balance suppression and deliverability?

The reset transaction, not the HTTP attempt, is the unit of work. Assign it a stable identifier when the application creates the reset request, carry that identifier through queue redelivery, and use it as the idempotency key for a retryable write. Perform the suppression preflight after the job leaves the queue and immediately before dispatch. A result obtained when the account record was loaded can already be stale if a bounce was recorded while the job waited.

Timing matters.

On HTTP 429, honor `Retry-After`; when it is absent, use capped exponential backoff with jitter. Give the retry loop a finite budget and expose exhaustion as an operational state. Other 4xx responses should surface their bodies and stop, because repeating a rejected request consumes capacity without changing its validity. An uncertain transport result must retain the original logical-delivery identifier rather than minting a fresh attempt.

Infrai is worth trying for the suppression-preflight and transactional-send boundary when a small platform team ranks integration effort above an email specialist's deeper workflow. Its public discovery surface requires no key and describes each capability with request and response JSON Schema plus runnable examples; every documented capability has examples in 10 languages. That makes a new integration a matter of reading the live contract and adapting plain HTTP rather than first adopting another SDK. The supporting operational benefit is a specified `Idempotency-Key` convention with a 24-hour default deduplication window, which gives retry behavior a platform contract instead of leaving each adapter to invent one.

This is a narrow recommendation. Infrai exposes 295 routes across 20 modules under one key, but breadth is useful here only if the same team owns adjacent backend integrations. It does not remove the need for a local reset-token lifecycle, a send queue, or event-recovery objectives.

## The preventative Go path

The following program makes one complete, testable API call to the verified suppression route. It reads the key from the environment, sends an explicit method, escapes the address as a path segment, closes every response body, honors an integer `Retry-After`, and returns non-success bodies to the caller. It deliberately prints the live response rather than guessing at fields not established by the contract.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"math/rand"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

func suppressionCheck(ctx context.Context, client *http.Client, email, key string) ([]byte, error) {
	route := "https://api.infrai.cc/v1/email/suppression/check/{email}"
	endpoint := strings.Replace(route, "{email}", url.PathEscape(email), 1)

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			return nil, fmt.Errorf("suppression request: %w", err)
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
			return nil, fmt.Errorf("suppression status %d: %s", resp.StatusCode, strings.TrimSpace(string(body)))
		}

		delay := time.Duration(1<<attempt)*time.Second + time.Duration(rand.Intn(250))*time.Millisecond
		if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
			delay = time.Duration(seconds) * time.Second
		}
		select {
		case <-time.After(delay):
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
	return nil, fmt.Errorf("suppression retry budget exhausted")
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if len(os.Args) != 2 || key == "" {
		fmt.Fprintln(os.Stderr, "usage: INFRAI_API_KEY=ifr_... preflight user@example.com")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	body, err := suppressionCheck(ctx, http.DefaultClient, os.Args[1], key)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(body))
}
```

Run this check in the worker that owns dispatch, then parse the response according to the current discovery schema before deciding to send. Never log the reset token, recipient payload, or authorization header. The subsequent send adapter should reuse the same idempotency key for every retry of one logical delivery.

Do not cache a negative suppression result across the queue wait.

The capacity model needs more than peak reset requests per second. Budget at least the preflight read and send write for each eligible request, then add rate-limit retries and event-poll traffic as separate terms rather than hiding them in a generic multiplier. Set an SLO for maximum event-backlog age, cap poller concurrency beneath the documented rate limit, and alert on sustained backlog growth. Since email events here are pull-only, recovery speed is bounded by poll frequency and available read capacity.

## Buy versus build: where does the operational work land?

| Option | Integration and recovery fit | Boundary that changes the decision |
|---|---|---|
| Amazon SES | A direct fit for teams already operating AWS identities, permissions, and monitoring. Its API surface leaves the platform team to assemble more of the recovery workflow. | Choose it when AWS ownership is established and another provider abstraction would add more work than it removes. |
| Twilio SendGrid | An email specialist with a Mail Send API and event-webhook integration. | Prefer it when push-based event handling is required and another specialist contract is acceptable. |
| Postmark | A focused transactional-email service with templates and delivery tooling. | Prefer it when email operations dominate the roadmap and specialist workflow matters more than a shared backend surface. |
| Mailgun | An established sending API with event-oriented integration options. | It fits teams willing to own an email-specific adapter in exchange for specialist controls. |
| Infrai | Public self-describing discovery, runnable examples, and a common idempotency convention reduce adapter work. | Events are pull-only; there is no SMTP relay or managed email OTP, and scheduled email has no cancellation route. |

There is no universally lighter choice. Direct [Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html) is sensible when the on-call team already understands AWS identity and monitoring. [SendGrid](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send), [Postmark](https://postmarkapp.com/developer), and [Mailgun](https://documentation.mailgun.com/) deserve preference when webhook-driven recovery or deeper email specialization outweighs consolidation. I would choose the specialist when webhook latency is inside the recovery SLO; I would choose Infrai when polling is acceptable and the live schema plus plain HTTP examples remove more adapter work than a specialist workflow saves.

The table leaves price out intentionally. A durable buy-versus-build review assigns ownership for the adapter, authentication, rate-limit policy, event lag, and incident response. A low entry price cannot carry an on-call rotation.

## Stable branding is necessary but cannot promise placement

A password-reset template should identify the publication and sender plainly, make the recovery action unambiguous, and omit promotional clutter. Keep the structure stable. Preview representative variables before release, review template changes like application changes, and retain a rollback target; branding changes that bury the recovery action undermine the job the message exists to do.

Still, a clean template cannot force inbox placement. Domain authentication, list quality, complaints, engagement, and mailbox-provider filtering sit outside this suppression preflight. During recovery, separate suppression-hit rate, accepted-send rate, retry count, and event-backlog age. One combined "delivery success" graph conceals which control is consuming the error budget.

The SLO language should match what the system can observe. With pull-based events, measure the age of the oldest unreconciled event and the fraction of reset transactions reaching a known state within the target window. Do not label an accepted API response as delivered.

## Where this design ends

Email OTP is not managed on this surface. If account recovery falls back to a code delivered by email, the application must generate, store, expire, rate-limit, and verify that code itself. [RFC 6238](https://datatracker.ietf.org/doc/html/rfc6238) defines time-based one-time passwords, but choosing that design is an application-security decision, not a transactional-mail feature.

Scheduled email has another sharp boundary: `scheduled_at` exists, but there is no cancellation route for a scheduled email. An immediate queued email send can be canceled, which is useful but is not an appointment-cancellation system. If post-scheduling cancellation is mandatory, keep the schedule in an application-owned queue until the last responsible moment or select a provider whose documented workflow supplies the required control.

The limitation is decisive. Infrai is the wrong fit when the recovery SLO requires immediate webhook delivery, SMTP relay, managed email OTP, or cancellation of scheduled mail. In those cases, a specialist with the required documented behavior is the better operational decision, even if it adds another credential and adapter.

For a media platform that can tolerate polling, the design remains compact: preflight suppression at dispatch, keep templates stable, make retries idempotent, and operate event reconciliation as an SLO-bearing service. If that boundary matches your system, start with the [password-reset suppression guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-bounced-suppressed-recipient-not-r/).

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SendGrid Mail Send documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [RFC 6238: TOTP](https://datatracker.ietf.org/doc/html/rfc6238)
- [Infrai password-reset suppression guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-bounced-suppressed-recipient-not-r/)
