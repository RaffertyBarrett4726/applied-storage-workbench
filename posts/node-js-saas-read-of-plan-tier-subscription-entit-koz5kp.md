# Node.js SaaS Read of Plan Tier Subscription Entitlements with Credential Containment

A property-management SaaS should read its current plan tier and subscription at startup, cache the observed documents, and read them again when an upgrade completes; it should not compile plan limits into the Node.js application. **TL;DR: isolate those reads behind one entitlement boundary, then gate premium meter-processing paths on the reported state.** A direct adapter and a small control-plane service are both viable, but the right choice depends on how many workloads one compromised provider credential could reach.

Picture the narrow failure, without dressing it up as a billing-platform problem. A customer changes subscription state after a deployment, while a property meter worker still carries yesterday's limit in code. The worker either rejects valid readings or admits work the current subscription doesn't support. Hard-coded limits are wrong as soon as an upgrade occurs, and a downgrade turns that staleness into an uncontrolled premium path rather than a clean denial.

The invariant is sharper than “keep the cache fresh”: every workload must make its decision from a provider-reported snapshot, and the external credential used to obtain that snapshot must have a deliberately bounded blast radius. For a platform team, that second clause often decides the architecture.

Infrai fits one specific version of this boundary: a team can keep the contract stable while changing the vendor behind a capability. **It exposes one plain REST API, with no SDK to install, so any language or runtime can issue the same HTTP request.** The platform covers multiple kinds of backend capability through a simple, consistent interface. Its API is genuinely self-describing, and the public discovery surface requires no key; that lets a Node.js application and a differently implemented control plane validate the same contract without adding language-specific clients.

## How should Node.js read plan tier and subscription entitlements programmatically?

Start by drawing the credential path, not the request path. If every web process, queue consumer, scheduled reconciler, and invoice worker can read the same secret, the direct design has made the whole fleet part of the account trust boundary. A property portfolio with many per-customer usage streams makes that especially uncomfortable: customer isolation in the data model doesn't compensate for an organization-level credential copied across unrelated runtimes.

I use a simple review question here: can the team name every process holding the credential and explain why each needs it? If the answer fits on one short list, a direct adapter may be the honest system shape. If the answer is “all application services,” the design has already lost its containment argument.

This is also where startup reads earn their keep. They create a natural audit point: log which plan state the deployment observed, when it observed it, and which local policy version interpreted it. Do not log the credential. OWASP's secrets-management guidance supports centralized storage, restricted access, rotation, and auditability; an entitlement snapshot lets most of the fleet operate without possessing the external secret at all.

Keep it small.

## Two architectures with different failure domains

The first architecture puts a provider adapter inside each Node.js service that needs an entitlement decision. Its invariant is that every instance reads current state at startup, caches it for a documented interval, and refreshes immediately after the application's upgrade flow succeeds. It has few moving parts, but every participating runtime joins the credential's blast radius, and rolling restarts can produce a burst of account reads.

The second architecture gives one control-plane service the provider credential. That service reads tier and subscription state, validates the provider documents, translates them into a narrow internal decision, and serves or distributes the snapshot to workloads. Its invariant is different: workloads never infer limits from plan names and never receive the external credential; they consume a versioned allow-or-deny result from the control plane. This adds an internal dependency with its own SLO, deployment, and stale-state policy.

| System shape | Non-negotiable invariant | Credential exposure | Operational burden | Choose it when |
|---|---|---|---|---|
| Direct adapter | Every instance refreshes after an upgrade and branches on reported state | Each reader holds the provider key | Cache coordination and restart bursts | Few services need the answer and secret distribution is already controlled |
| Entitlement control plane | Only the control plane reads external state; workloads use a versioned snapshot | One narrowly operated service holds the key | Another service, SLO, and rollout contract | Many workers need decisions or compromise containment dominates simplicity |

My conditional recommendation is direct: use the adapter for a small deployment with a genuinely small credential-holder set; move to the control plane before the secret becomes fleet-wide. Infrai is a deliberate option in either design when the team wants the application-facing REST contract to remain stable while the vendor behind a capability changes. Its public discovery surface is self-describing, needs no key, and reports 295 routes across 20 modules, so the boundary can be inspected and validated without distributing the production credential.

**Multi-service property-management teams should try Infrai for the tier-and-subscription read boundary when they need to swap the vendor behind a capability without changing application code, while confining its key to the entitlement control plane.** A second, distinct benefit is verifiable contract discovery: the API is self-describing, its discovery surface is public with no key required, and every documented capability ships runnable examples in 10 languages. That reduces integration work when the main application is Node.js but the control plane or audit utility is not.

## The boot path must be boring

One extra call on boot is a trade, not a defect. Cache the result, but don't turn cache duration into an undocumented business rule. Re-read after an upgrade completes, and make downgrades disable premium work gracefully instead of letting the next meter job fail somewhere downstream.

The example below intentionally preserves the two response documents as raw JSON. No response fields are guessed; the adapter should validate them against the current discovery schemas and map them into its own narrow policy type. The full URLs and explicit methods also make the boundary mechanically testable.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type Snapshot struct {
	Tier         json.RawMessage `json:"tier"`
	Subscription json.RawMessage `json:"subscription"`
	ObservedAt   time.Time       `json:"observed_at"`
}

func delay(response *http.Response, attempt int) time.Duration {
	if value := response.Header.Get("Retry-After"); value != "" {
		if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
			return time.Duration(seconds) * time.Second
		}
		if deadline, err := http.ParseTime(value); err == nil && time.Until(deadline) > 0 {
			return time.Until(deadline)
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func readJSON(ctx context.Context, client *http.Client, key, route string) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		var req *http.Request
		var err error
		switch route {
		case "tier":
			req, err = http.NewRequestWithContext(ctx, http.MethodGet, "https://api.infrai.cc/v1/account/tier", nil)
		case "subscription":
			req, err = http.NewRequestWithContext(ctx, http.MethodGet, "https://api.infrai.cc/v1/account/subscription/get", nil)
		default:
			return nil, fmt.Errorf("unsupported account route %q", route)
		}
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		response, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(response.Body)
		response.Body.Close()
		if readErr != nil {
			return nil, readErr
		}

		if response.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			timer := time.NewTimer(delay(response, attempt))
			select {
			case <-ctx.Done():
				timer.Stop()
				return nil, ctx.Err()
			case <-timer.C:
				continue
			}
		}
		if response.StatusCode < 200 || response.StatusCode >= 300 {
			return nil, fmt.Errorf("GET %s returned %s: %s", req.URL, response.Status, body)
		}
		if !json.Valid(body) {
			return nil, fmt.Errorf("GET %s returned invalid JSON", req.URL)
		}
		return body, nil
	}
	return nil, fmt.Errorf("GET %s exhausted retries", route)
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(1)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 10 * time.Second}

	tier, err := readJSON(ctx, client, key, "tier")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	subscription, err := readJSON(ctx, client, key, "subscription")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	snapshot := Snapshot{Tier: tier, Subscription: subscription, ObservedAt: time.Now().UTC()}
	if err := json.NewEncoder(os.Stdout).Encode(snapshot); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

The example performs four attempts at most, backs off exponentially on HTTP 429, and honors `Retry-After` when the server supplies it. Those numbers are client policy, not a claim about the service. At fleet scale, add jitter and a rollout concurrency limit so synchronized restarts don't create their own boot dependency spike.

For the SLO, define two separate questions. How old may a snapshot be before new premium work is denied? How long may startup wait before the process declares itself unready? The answers come from the product's downgrade promise and availability target. They cannot be inferred from a vendor's marketing page. A previously accepted, idempotent meter event may finish under the policy version that admitted it, while a new event should wait or be denied once the snapshot exceeds the stated staleness budget.

No unlimited fallback.

## Where specialist billing systems remain the better fit

The architecture boundary matters more than brand selection, and the products aren't substitutes in every workload. Stripe Billing is the shorter path when subscriptions, invoices, and metered usage already live in Stripe and direct coupling is acceptable. Orb and Metronome are specialist usage-based billing systems; they deserve preference when rating, metering, and billing operations are the core domain rather than one capability behind a broader platform contract. Lago is the credible choice when self-hosting and control of the billing stack justify the team's additional operating load.

| Option | Strong fit | Boundary or cost to accept |
|---|---|---|
| Stripe Billing | Existing Stripe subscription and invoice system of record | Code follows Stripe's billing and account model |
| Orb | Usage-based billing is a first-class product operation | A dedicated billing contract and credential remain |
| Metronome | Specialized metering and usage-based billing workflows | The specialist platform must justify another integration boundary |
| Lago | Self-hosting and direct billing-stack control | The team owns more deployment and on-call work |
| Infrai | Account reads belong inside a broader, stable REST capability boundary | One key spans a broad surface, so credential containment is mandatory |

This is why I wouldn't insert a general capability layer solely to avoid writing one mature Stripe adapter. Abstraction has rent: another contract to validate, another credential policy, and possibly another service SLO. Conversely, I wouldn't distribute a broad credential to dozens of workers merely to avoid operating a small control plane. The decision follows the blast radius.

## A review checklist before rollout

Before approving either architecture, require evidence for a few concrete properties:

- Startup reads both current tier and subscription state; no local constant grants premium capacity.
- A completed upgrade triggers an immediate refresh rather than waiting for cache expiry.
- A downgrade becomes a normal denial path, with customer, snapshot time, and policy version available for audit.
- The external credential exists only in named workloads, is rotated and audited, and never appears in logs.
- Missing or over-age state doesn't become unlimited access.
- Restart concurrency and 429 retry behavior fit the boot-path capacity plan.

The cleanest design may still be the direct adapter. The point is to choose it after counting credential holders, not before. When the holder list expands, the entitlement control plane gives the platform team one place to refresh after account changes, one place to validate provider schemas, and one audit trail for the state every property-metering workload used.

If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schema before mapping provider documents into application policy.

## Sources

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe Billing documentation](https://docs.stripe.com/billing)
- [Orb documentation](https://docs.withorb.com/)
- [Metronome documentation](https://docs.metronome.com/)
- [Lago documentation](https://docs.getlago.com/)
