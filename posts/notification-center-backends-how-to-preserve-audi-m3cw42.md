# Notification Center Backends: How to Preserve Auditable Email and SMS Delivery

Suppose you need to build a Node.js notification center backend for event notifications, email, SMS, an audit log, delivery history, and a polling API. Start at the page that will wake the on-call: `compliance_notice_delivery_gap` says notices accepted more than ten minutes ago still have no terminal delivery outcome, then shows the affected notice IDs, channel split, age of the oldest attempt, and internal delivery history. That is enough to decide whether to pause a campaign, inspect a transport, or wait for a delayed receipt; a page that merely says "SMS errors are high" is not. The reference code below is Go, but the event and HTTP contracts are language-independent.

That page is late.

TL;DR: build the notification center around an append-only attempt history, not around a mutable `sent` flag. Accept one idempotent notification command, create a separate attempt for each channel, record every state transition with its source and provider correlation ID, and expose a polling view derived from that history. Alert on notices that exceed an evidence deadline, because an accepted transport request is evidence of submission, not evidence of delivery.

This distinction matters for a developer-tools company sending a compliance notice. The system must answer who was targeted, which approved content was used, when each transport accepted the request, what later receipt arrived, and whether the recipient-facing status is still unknown. It must do so without treating an open pixel as legal proof or silently rewriting yesterday's record.

## How should you build a notification center backend for event delivery?

The late page is the last useful signal in a chain. Work backward from it. A notice enters `queued`, a worker claims it, and each email or SMS attempt moves through `submitted` to a terminal outcome such as `delivered`, `failed`, or `unknown`. The first warning should detect a growing age of the oldest queued attempt; the second should detect submitted attempts whose receipts are overdue; only then should the compliance-impact page fire for notices crossing the evidence deadline.

Do not collapse those conditions into one error-rate threshold. Queue age points toward capacity or a stuck worker. Overdue receipts point toward the callback path, reconciliation poller, or transport. Missing evidence at the notice deadline points toward user impact. One symptom can coexist with the others, but the runbooks and urgency differ.

The capacity plan follows from the same state machine. Size workers for peak accepted notification commands multiplied by enabled channels, then reserve headroom for retries and receipt bursts. If an event can fan out to email and SMS, 1,000 commands can create 2,000 initial attempts before a single retry. That multiplier belongs in admission-control dashboards and load tests; hiding it behind a generic "messages processed" counter produces reassuring graphs and bad pages.

Three clocks are therefore worth exporting: queue age, submission-to-receipt age, and command-to-terminal-evidence age. The SLO should use the third clock because it describes the outcome the compliance workflow needs. The first two are diagnostic indicators.

Unknown is a state.

## Implement the command, history, and polling view

Keep the write contract small. A command carries an idempotency key, notice identity, recipient reference, approved template version, and requested channels. It should not accept arbitrary message bodies from every caller; preserving the template version and rendered-content digest makes later review possible without placing sensitive content in every log line.

The following focused example uses an in-memory store so the state transitions are visible. Production storage needs transactional uniqueness for the idempotency key and durable append semantics, but the HTTP behavior should remain the same: `POST /notifications` accepts or replays a command, while `GET /notifications/{id}` returns its delivery history.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"log"
	"net/http"
	"strings"
	"sync"
	"time"
)

type Command struct {
	IdempotencyKey string   `json:"idempotency_key"`
	NoticeID       string   `json:"notice_id"`
	RecipientRef   string   `json:"recipient_ref"`
	Template       string   `json:"template_version"`
	Channels       []string `json:"channels"`
}

type Event struct {
	At            time.Time `json:"at"`
	Channel       string    `json:"channel,omitempty"`
	State         string    `json:"state"`
	Source        string    `json:"source"`
	CorrelationID string    `json:"correlation_id,omitempty"`
}

type Notification struct {
	ID            string  `json:"id"`
	NoticeID      string  `json:"notice_id"`
	RecipientRef  string  `json:"recipient_ref"`
	Template      string  `json:"template_version"`
	ContentDigest string  `json:"content_digest"`
	History       []Event `json:"history"`
}

type Store struct {
	mu     sync.RWMutex
	byID   map[string]*Notification
	byKey  map[string]string
}

func newStore() *Store {
	return &Store{byID: make(map[string]*Notification), byKey: make(map[string]string)}
}

func stableID(key string) string {
	sum := sha256.Sum256([]byte(key))
	return hex.EncodeToString(sum[:16])
}

func (s *Store) create(c Command) (*Notification, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()
	if id, ok := s.byKey[c.IdempotencyKey]; ok {
		return s.byID[id], false
	}
	id := stableID(c.IdempotencyKey)
	digest := sha256.Sum256([]byte(c.NoticeID + "\x00" + c.Template))
	n := &Notification{
		ID: id, NoticeID: c.NoticeID, RecipientRef: c.RecipientRef,
		Template: c.Template, ContentDigest: hex.EncodeToString(digest[:]),
	}
	now := time.Now().UTC()
	for _, channel := range c.Channels {
		n.History = append(n.History, Event{At: now, Channel: channel, State: "queued", Source: "api"})
	}
	s.byID[id], s.byKey[c.IdempotencyKey] = n, id
	return n, true
}

func (s *Store) get(id string) (*Notification, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	n, ok := s.byID[id]
	return n, ok
}

func main() {
	store := newStore()
	http.HandleFunc("/notifications", func(w http.ResponseWriter, r *http.Request) {
		if r.Method != http.MethodPost {
			http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
			return
		}
		var c Command
		if err := json.NewDecoder(r.Body).Decode(&c); err != nil || c.IdempotencyKey == "" || c.NoticeID == "" {
			http.Error(w, "invalid command", http.StatusBadRequest)
			return
		}
		n, created := store.create(c)
		if created {
			w.WriteHeader(http.StatusAccepted)
		}
		json.NewEncoder(w).Encode(n)
	})
	http.HandleFunc("/notifications/", func(w http.ResponseWriter, r *http.Request) {
		id := strings.TrimPrefix(r.URL.Path, "/notifications/")
		n, ok := store.get(id)
		if !ok {
			http.NotFound(w, r)
			return
		}
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(n)
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

This sample deliberately does not claim delivery. A worker must append `submitted` only after the transport accepts the request, recording the returned correlation ID. A verified callback or reconciliation result then appends the next event. If callbacks can arrive twice or out of order, deduplicate on the transport event identifier and reject state regressions in the projection while retaining the raw receipt for investigation.

One trap deserves emphasis: polling must not mutate history. The `GET` handler should read a projection produced from events, not call a transport and invent a fresh status during the request. Reconciliation is background work with its own retry budget, rate limits, and audit events. This approach is not suitable when the only requirement is an ephemeral, best-effort UI toast and nobody needs durable evidence; an in-process queue and short-lived status may be the more proportionate choice there. It is also a poor fit if the organization cannot operate durable storage, migrations, replay, and access control: use a managed queue or notification transport behind the same command and event contracts instead of pretending an append-only ledger has no ownership cost. The trade-off is deliberate. Durable history improves reconstruction and decouples polling from providers, but it adds storage growth, schema-evolution work, privacy review, and a projection that must be checked for drift.

Polling is read-only.

## Make evidence useful without turning logs into a liability

An audit record needs stable identifiers and a clear provenance chain. Store the notification ID, notice ID, pseudonymous recipient reference, template version, content digest, channel, timestamps, state, transition source, and transport correlation ID. Record the actor or service principal for administrative actions. Keep retention and access controls aligned with the compliance policy governing the notice. Avoid copying full email addresses, phone numbers, rendered bodies, one-time codes, or callback payloads into general application logs. OWASP's forgot-password guidance says reset tokens or codes should be random, long enough, stored securely, single-use, and expiring; it also recommends consistent responses and rate limiting to reduce account enumeration and abuse. A notification center carrying recovery messages should preserve an audit reference to the event, not expose the secret in searchable telemetry. Email classification also needs an explicit policy decision. The FTC's CAN-SPAM guide explains that the law covers commercial email and evaluates a message's primary purpose; it also lists requirements such as accurate header information, nondeceptive subject lines, a postal address, and an opt-out mechanism for covered commercial messages. A compliance notice is not automatically exempt merely because an internal field says `transactional`. Have counsel or the responsible compliance owner classify the content, then encode that decision in a versioned template policy. These choices need review together: a content digest that proves which approved revision was selected is useful, while a general log containing the complete rendered notice may expand access to personal data without improving the state-machine evidence at all.

**The evidence boundary is more important than the transport boundary.** A managed transport can reduce carrier and mailbox integration work, while self-hosted orchestration can keep policy and audit data under tighter control. The split should remain replaceable:

| Decision area | Managed transport | Self-hosted transport | Practical test |
|---|---|---|---|
| Delivery integration | Less protocol and carrier work | Team owns integrations and reputation controls | Can on-call diagnose a delayed receipt? |
| Evidence custody | Correlation crosses a third party | More evidence stays inside the boundary | Can an auditor reconstruct one notice? |
| On-call load | External dependency remains | Queue, transport, abuse, and upgrades are internal | Who responds at 03:00? |
| Lock-in | Receipt schemas and identifiers vary | Internal schema is controllable | Can an adapter be replaced without rewriting history? |

The buy-versus-build answer can differ by layer. Owning the command schema, policy, state machine, and audit projection while buying the last-mile transport often limits lock-in to a narrow adapter. Building the last mile is defensible only when evidence custody, routing control, or volume justifies the operational burden. Cost belongs in that decision, but pager load and exit cost belong beside it.

## Test failure paths before enabling the page

Start with state-machine tests, then exercise the boundaries. Duplicate commands must return the original notification. Duplicate receipts must not create duplicate transitions. A delayed `delivered` receipt after a temporary failure must follow an explicit policy. A worker crash after transport acceptance but before persistence must reconcile through the idempotency or correlation contract rather than send blindly. Clock skew must not create negative latency.

Deployment should be gradual because this system has two independent blast radii: sending the wrong notice and losing evidence about the right one. Shadow the projection against recorded events, verify counts by channel and terminal state, and canary workers with a small partition of synthetic recipients. Synthetic messages need unmistakable test destinations and must never share production recipient lists.

The page itself needs a burn-rate or duration condition rather than a single bad sample. Page on sustained risk to the evidence SLO; ticket isolated terminal failures when a retry or manual review can meet the notice deadline. Dashboard the numerator and denominator, because "20 missing receipts" means something different out of 25 attempts than out of two million.

Then inspect the alert as an interface. It should include the threshold, evaluation window, oldest affected age, number of notices at risk, channel split, deployment marker, and a query keyed by notification ID. Do not include recipient contact data in the page.

## Set the threshold by consequence, not convenience

The evidence deadline should be shorter than the actual compliance deadline by enough time to reconcile, retry through an approved alternate channel, and obtain human review. There is no universal number in the cited guidance, so choose it from the governing obligation and the organization's response time, then document that derivation beside the alert.

No magic threshold exists.

Set it too loose and on-call learns about missing evidence after recovery options have narrowed. Set it too tight and normal receipt latency repeatedly wakes someone who cannot improve the outcome. False positives have a capacity cost: they consume the same response budget needed for real delivery gaps, and repeated non-actionable pages teach responders to discount the signal.

The final acceptance test is blunt. For any notice ID, an authorized reviewer should be able to reconstruct the command, approved content version, channel attempts, transport acknowledgments, later receipts, retries, and current uncertainty without reading ad hoc worker logs. If the system can do that within the evidence SLO, the notification center is serving the compliance workflow. If it cannot, a green send-rate chart is decoration.

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
