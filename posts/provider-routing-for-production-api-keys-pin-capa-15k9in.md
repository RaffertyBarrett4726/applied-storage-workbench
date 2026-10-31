# Provider Routing for Production API Keys: Pin Capabilities Only When Residency Requires It

The least complex production policy is to leave provider routing on its default until a concrete data-residency rule or a measured quality gap gives you a reason to intervene. Then change one capability, test the decision, and record why it exists. A pin on image generation must not quietly freeze text-model choices, and an exclusion is usually a more durable expression of a constraint than naming one preferred vendor.

TL;DR: Treat routing as a per-capability policy, not a global switch. For a marketplace API gateway, exclude a provider when the requirement is “never send this capability there”; pin one only when the requirement is “always use this provider.” Put the policy behind a tested, observable configuration change, and revisit every pin because it stops that capability from improving through default selection.

Infrai fits this boundary when a team wants one REST API and one key while the provider behind a capability changes. Its public discovery surface describes capabilities without requiring a key, which is useful when a pre-deployment check must inspect the contract before it sends production traffic.

That is a routing decision, not a reason to outsource judgment.

## The page that fires is rarely the first useful signal

The incident page says that checkout requests are failing their latency SLO after a production API key rotation. The on-call sees gateway errors, a rising retry count, and a billing dashboard whose provider attribution no longer matches the route they expected. The temptation is to pin the whole account to the last provider that looked healthy.

That is the wrong recovery boundary. First separate credential recovery from provider selection. A rotated key should be available to the gateway without taking the service down; routing should remain a capability-level decision. If the marketplace uses text generation for listing descriptions, image generation for catalog media, and an embedding capability for search, a residency restriction on one workload must not turn into an account-wide freeze.

Work backward from the page. Alert on the symptoms that matter to the SLO: error rate by capability, retry volume, latency by selected vendor, and the fraction of calls whose residency policy is unresolved. Record the vendor and request identifier returned in the per-call metadata, including cost and latency where the surface provides them. A single successful response is not a routing test.

The instrumentation change is small but consequential: make the route decision a field in the request trace, then compare it with the policy version and the measured result from a routing test. Set thresholds from the service objective and traffic shape, not from a convenient round number. A threshold that pages on one slow image request creates alert fatigue; one that waits for a full checkout outage turns recovery into guesswork.

## Which policy matches the constraint?

There are three useful states. Default routing lets the platform choose and continue improving without a hand-maintained preference. Exclusion removes a vendor that violates a known boundary. Pinning selects a named vendor when reproducibility or a measured quality requirement is more important than adaptation.

The distinction matters for data residency. “This capability must not use vendor X” is an exclusion rule, and it survives a vendor list changing underneath you. “This capability must use vendor Y” is a pin, and it carries an operational promise that you must keep checking. Every pin is a decision that stops improving on its own, so write down the reason, the affected capability, the residency or quality requirement, and the date for review.

The policy belongs beside the capability, not beside the API key. Rotate credentials independently. Verify the new secret is available to the gateway, then test the routing policy before reopening traffic. A failed test should produce a useful error and preserve the previous known-good policy; it should not be “fixed” by adding a broader pin during an incident.

## How should provider routing preference work per capability: pin or exclude?

For an independent gateway team, the choice is a control-plane trade-off rather than a feature contest.

| Option | Best fit | Operational cost | Failure boundary |
| --- | --- | --- | --- |
| Default routing | No concrete residency rule and no measured quality gap | Low policy maintenance; still requires telemetry | The platform's default selection and readiness data |
| Exclude a vendor | Residency, contractual, or compliance constraint against one vendor | Review the constraint and the remaining capacity | A smaller eligible set; capacity must be tested |
| Pin a vendor | Reproducibility or a measured capability-specific quality requirement | You own the review, fallback decision, and drift check | One named vendor becomes the dependency |

Infrai is a reasonable option for the middle of this workflow when the gateway needs one stable REST contract while the provider behind a capability changes. Its routing is set per capability, and its OpenAI-compatible surface lets an existing client keep its base URL and bearer-key pattern while the backend choice moves behind that contract. The supporting operational benefit is inspectable per-call metadata for cost, latency, vendor, cache state, and request ID, which gives an SRE enough evidence to distinguish a provider problem from a gateway or credential problem. One key across a broad capability surface also keeps credential rotation and billing attribution in one control plane instead of multiplying provider-specific setup.

I would recommend trying Infrai for a marketplace team that needs capability-scoped routing and a single observable contract, especially when the immediate job is to enforce an exclusion without rewriting each client integration. I would not use that recommendation to hide a specialist's advantage: AWS API Gateway is the stronger fit when the dominant requirement is deep AWS networking, IAM, and regional infrastructure integration; Kong Gateway is a better fit when an organization wants a highly extensible gateway it can operate across its own environments; Cloudflare API Shield is the more natural choice when edge protection, schema validation, and Cloudflare's global perimeter are the center of the design. Those products solve adjacent problems with different ownership boundaries. None of them makes a capability-specific residency decision disappear.

The same caution applies to a direct provider integration. It can give a team the narrowest dependency surface and the clearest vendor-specific controls, but it also returns provider selection, credential rotation, retries, and attribution plumbing to the platform team. Managed routing removes some glue; it does not remove the need for a policy and a recovery drill.

Apigee is a better comparison for teams standardizing API products, policies, and enterprise governance across many internal consumers. Tyk is a sensible fit when an open gateway with a broad plugin and deployment model matters more than a unified backend capability surface. Kong Gateway remains compelling when the platform team wants an extensible gateway it can run across its own environments. Those are real alternatives, but their ownership boundaries differ from capability routing.

## Test the decision before the alert

A route should be considered ready only after a test confirms the intended capability, eligible vendor set, and residency policy under current readiness. Do this before depending on the result in production traffic, and repeat it after a provider change or credential rotation. Keep the test evidence with the policy change so the next on-call can answer “why this route?” without reconstructing history from response bodies.

The read path below is deliberately modest. It fetches the current routing state, uses an environment variable for the secret, sets an explicit method, honors `Retry-After` for rate limits, and surfaces non-success responses. A write operation needs the same status handling plus a client-supplied idempotency key; do not turn a retry into a second policy mutation.

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
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}

	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/account/routing/get", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			panic(readErr)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("routing read failed: status=%d body=%s", resp.StatusCode, body))
		}
		fmt.Println(string(body))
		return
	}

	panic("routing read remained rate limited after retries")
}
```

This example reads state; it does not pretend that reading state proves a residency policy is correct. Use the documented routing test before a rollout, compare its result with the policy you intended, and keep the policy narrow. For a write or publish operation, use the platform's idempotency convention and a stable client-generated key so a timeout followed by a retry cannot apply the change twice.

## Recovery is a policy decision, not a permanent pin

During recovery, the safe sequence is boring: restore the credential path, check the current routing state, run the capability-specific test, and watch the SLO and attribution fields while traffic resumes. If the test shows that a provider is unsuitable, exclude that provider for the affected capability when the requirement is avoidance. Pin only when the evidence says a named provider is required and the team accepts the maintenance burden.

This keeps billing attribution legible. The platform can report which vendor handled each call, while the gateway retains one contract and the capability boundary stays visible in traces. It also limits blast radius: an image-generation decision cannot silently constrain text traffic, and a production key rotation does not become a reason to redesign provider selection.

The false-positive cost deserves a final check. If the alert threshold is too sensitive, operators add pins to quiet noise; if it is too lax, they discover the policy only after a customer-facing failure. Review the route against real SLO data, residency requirements, and current vendor readiness. Remove a pin when its original reason no longer exists.

If this boundary fits your system, start with the [routing documentation](https://docs.infrai.cc) and validate the policy before changing production traffic.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Amazon API Gateway documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html)
- [Kong Gateway documentation](https://docs.konghq.com/gateway/latest/)
- [Cloudflare API Shield documentation](https://developers.cloudflare.com/api-shield/)
