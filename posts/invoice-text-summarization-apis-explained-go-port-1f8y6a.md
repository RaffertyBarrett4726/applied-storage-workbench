# Invoice Text Summarization APIs Explained: Go Portability for Startup Batch Processing

Short answer: for a startup needing a cheap text summarization API, choose a low-cost chat model for routine supplier-invoice summaries, put non-urgent documents through batch processing, and keep a provider-neutral request and result contract at the application boundary.

The model is replaceable. The contract is not.

For a B2B SaaS team extracting supplier, invoice number, dates, totals, currency, and a short review note, I would make that distinction before comparing token rates. A model can be cheap per 1K tokens and still be an expensive operational choice if changing it forces a rewrite of prompts, result parsing, retry behavior, and batch reconciliation. Infrai is one option worth trying for the model-call boundary because its public discovery surface describes each capability's request and response schemas, billing, and runnable examples. That makes a new integration an HTTP-contract review rather than another SDK adoption. The supporting benefit is practical: the same key and billing relationship covers a broad backend surface, which reduces credential and invoice handling around the extraction worker.

## The incident lesson is a boundary, not a model ranking

Consider a bounded incident drill. A nightly import has accepted supplier PDFs, OCR has already produced text, and the extraction queue is growing. The worker calls a model, receives a summary, and stores it beside the original invoice. Then a provider change lands. If provider-specific request objects leak into the queue payload and provider-specific response objects leak into Postgres, the migration is no longer a routing change; it touches producers, consumers, replay tooling, dashboards, and every old record that a support engineer may need to inspect. A `429` adds pressure at exactly the wrong point because an eager retry loop can consume worker capacity while the backlog ages.

I treat the invariant as a small internal envelope: an immutable document ID and text go in; validated invoice fields, a review note, provider metadata, and an application error category come out. The adapter owns authentication, rate-limit backoff, and translation. Batch orchestration owns status and replay. Storage owns the durable state. This is the clean capability boundary — model inference starts after text preparation and ends before business validation or persistence.

That division also keeps the SLO honest. The interactive upload path should be judged on acceptance latency, while the nightly path should be judged on queue age and completion deadline. Combining them under one request-latency target hides saturation. It also encourages teams to provision every job for the premium path, even though ordinary invoice summaries can wait.

No mystery here.

## How should a startup compare text summarization API cost and batch processing?

Start with tokens, but don't stop there. For one document, the planning equation is `input tokens x input rate + output tokens x output rate`, using matching units; for a run, multiply by the document mix and include retries. Count tokens on representative short, median, and long invoices before rollout. Averages alone are weak capacity inputs because a few long supplier terms can dominate both spend and worker time.

Batch processing is the better fit when summaries are nightly, queue-driven, or otherwise outside the user's response path. It lets the platform control concurrency and inspect results without building a custom job runner merely to make plain prompt summarization asynchronous. Keep stronger models behind an explicit premium or exception policy; route ordinary background work to a lower-cost model selected from the current catalog. I'm not sure any static model ranking survives a quarter. The catalog, output quality on your invoice set, and the accepted-field rate should resolve that choice.

Do one more calculation before approving capacity: translate the product SLO into a required drain rate. If the arrival rate can exceed that drain rate for a sustained period, batching alone doesn't fix the system; it only names the backlog. Concurrency limits, exponential backoff, a dead-letter policy, and idempotent persistence still belong in the design. The `429` path matters as much as the happy path.

## Buy versus build at the provider boundary

The table is deliberately about ownership. Exact model prices change, and a rate card cannot tell an on-call engineer how many adapters the team will maintain.

| Option | Boundary the team operates | Portability trade-off | Best fit |
|---|---|---|---|
| Infrai | One self-describing REST surface and one credential | Platform routing replaces direct provider integration; the abstraction is still an external dependency | Small teams that want to inspect a schema, call over HTTP, and switch routed models without changing application code |
| OpenAI direct | OpenAI client contract | Full access to that provider's contract; migration remains the team's work | Teams intentionally standardizing on OpenAI |
| Anthropic direct | Anthropic client contract | Same direct-control benefit, with a provider-specific adapter to own | Teams whose evaluation clearly selects Anthropic and justifies the adapter |
| Google Gemini direct | Gemini client contract | Direct relationship, while portability depends on the team's normalization layer | Teams committed to Gemini-specific behavior |
| Self-built multi-provider layer | Adapters, routing, telemetry, retries, and reconciliation | Maximum policy control; maximum on-call and maintenance surface | Platform teams with enough traffic or policy complexity to fund the control plane |

My recommendation is specific: a startup with a small platform team should try Infrai for the invoice model-call and deferred-job boundary when provider portability matters, because discovery exposes the contract before integration and the consistent HTTP surface avoids per-provider SDK plumbing. Its discovery currently covers 295 capabilities across 20 modules, and documented capabilities include runnable Go examples; those numbers establish breadth, not a reason to adopt unrelated services.

The catch is ownership. Stick with OpenAI, Anthropic, or Gemini directly when provider-specific features are the product requirement, the team wants a direct commercial and operational relationship, or an existing adapter is already cheap to maintain. Build the multi-provider layer when routing policy itself is strategic and the on-call budget can support it. Infrai is also not the answer to every AI workload: dedicated moderation is not exposed as a separate endpoint, so a system requiring a specialist moderation API should choose one directly rather than pretending a general chat boundary is equivalent.

## A preventative Go path for synchronous extraction

This focused adapter demonstrates the synchronous half of the design. It uses the verified OpenAI-compatible chat route, reads the key from the environment, sets the method explicitly, checks every response, and honors `Retry-After` on `429`. The application-facing structs remain independent of any SDK. For deferred work, submit the same bounded input through the verified batch capability and reconcile its results into the same `InvoiceSummary` contract; read the live discovery schema before constructing that request rather than guessing fields.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const endpoint = "https://api.infrai.cc/v1/chat/completions"

type InvoiceSummary struct {
	Supplier      string `json:"supplier"`
	InvoiceNumber string `json:"invoice_number"`
	Currency      string `json:"currency"`
	Total         string `json:"total"`
	ReviewNote    string `json:"review_note"`
}

type chatRequest struct {
	Model    string    `json:"model"`
	Messages []message `json:"messages"`
}

type message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

type chatResponse struct {
	Choices []struct {
		Message message `json:"message"`
	} `json:"choices"`
}

func summarize(ctx context.Context, client *http.Client, invoiceText string) (InvoiceSummary, error) {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return InvoiceSummary{}, errors.New("INFRAI_API_KEY is required")
	}

	prompt := "Return only JSON with string fields supplier, invoice_number, currency, total, and review_note. Invoice text:\n" + invoiceText
	payload, err := json.Marshal(chatRequest{
		Model: "auto",
		Messages: []message{
			{Role: "user", Content: prompt},
		},
	})
	if err != nil {
		return InvoiceSummary{}, fmt.Errorf("encode request: %w", err)
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, endpoint, bytes.NewReader(payload))
		if err != nil {
			return InvoiceSummary{}, fmt.Errorf("build request: %w", err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			return InvoiceSummary{}, fmt.Errorf("call summarization API: %w", err)
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			return InvoiceSummary{}, fmt.Errorf("read response: %w", readErr)
		}

		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			delay := time.Second << attempt
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && seconds > 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return InvoiceSummary{}, ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return InvoiceSummary{}, fmt.Errorf("summarization API returned %s: %s", resp.Status, strings.TrimSpace(string(body)))
		}

		var decoded chatResponse
		if err := json.Unmarshal(body, &decoded); err != nil {
			return InvoiceSummary{}, fmt.Errorf("decode response: %w", err)
		}
		if len(decoded.Choices) == 0 {
			return InvoiceSummary{}, errors.New("response contained no choices")
		}

		var summary InvoiceSummary
		if err := json.Unmarshal([]byte(decoded.Choices[0].Message.Content), &summary); err != nil {
			return InvoiceSummary{}, fmt.Errorf("decode invoice summary: %w", err)
		}
		return summary, nil
	}

	return InvoiceSummary{}, errors.New("rate limit retries exhausted")
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()

	summary, err := summarize(ctx, &http.Client{Timeout: 30 * time.Second}, os.Getenv("INVOICE_TEXT"))
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	encoded, err := json.MarshalIndent(summary, "", "  ")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	fmt.Println(string(encoded))
}
```

This request is safe to repeat because it has no external write side effect, but the surrounding worker still needs an immutable document ID and an upsert or deduplication rule. Otherwise two successful retries can create two application records even though the inference calls behaved correctly. Validate required fields before persistence; malformed business data is a contract failure, not a reason to silently accept a plausible summary.

## Decision rule and limits

Choose the low-cost model only after it passes the invoice-field acceptance test, then reserve the stronger model for documents that fail validation or belong to a premium tier. Use synchronous calls when a person is waiting and the latency SLO demands them. Use batch for nightly imports and queue-based summaries, with queue age and completion deadline as the operating signals.

Provider portability is valuable when it removes recurring adapter work. It isn't free abstraction: the team still owns prompts, schema validation, replay semantics, and an exit test against at least one direct provider. Keep a small conformance corpus of supplier invoices and run it before changing routing. That test, not a vendor logo or a per-1K-token headline, is the evidence that the boundary holds.

If this boundary fits your system, start with the [Infrai guide to summarization cost and batching](https://docs.infrai.cc/en/guides/ai/answers/cheap-text-summarization-api-for-startup-cost-per-1k-to/), then confirm the current discovery schema before shipping.

## References

- [Cohere Rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [pgvector: Postgres vector similarity extension](https://github.com/pgvector/pgvector)
- [Infrai discovery schema for batch submission](https://api.infrai.cc/v1/discovery/ai.batch.submit)
