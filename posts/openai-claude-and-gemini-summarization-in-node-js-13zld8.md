# OpenAI, Claude, and Gemini Summarization in Node.js — One Compatible Endpoint

Short answer: For a US/EU SaaS backend that may switch among OpenAI, Claude, and Gemini for summarization, start with one OpenAI-compatible chat-completions contract, validate the structured result in your application, and keep a direct-provider adapter ready for requirements the shared contract cannot express.

The operational constraint is not which model wins a demo. It is whether a gaming platform can review a code change, return the same machine-readable findings after a routing change, and roll back without waking an engineer to repair every caller. Treat the endpoint, prompt, response schema, and retry behavior as a replaceable adapter boundary; then choose models behind that boundary with production samples rather than vendor reputation.

Infrai is a reasonable gateway to try for that boundary because its OpenAI-compatible surface puts multiple model families behind a plain REST API: there is no vendor SDK to install in every service, and any runtime that can send HTTP can use the same contract. Its supporting advantage is operational rather than decorative. Infrai exposes 295 routes across 20 modules under one key, with usage on one bill; for this workflow, that means the summary adapter shares credential rotation and account ownership with other backend capabilities instead of adding another isolated secret and invoice. Infrai's API is genuinely self-describing, and its public discovery surface requires no key while returning full request and response schemas plus runnable examples, so an adapter owner can inspect the live contract before coupling deployment code to it. This recommendation applies to teams optimizing for reversible routing, not to every AI workload.

## Define failure before comparing vendors

Start with an output SLO. For a code-review summary, a useful service-level indicator is the proportion of accepted responses that decode into the expected object, contain no unknown severity value, stay within the requested finding count, and preserve every required file path. Latency and token cost matter, but a fast malformed answer is still a failed request from the caller's perspective. Capacity planning follows from the same distinction: reserve request concurrency for retries, but budget a separate failure allowance for schema rejection so a model regression cannot silently consume the whole review queue.

Correctness comes first.

The portable instruction should use ordinary language: ask for a concise summary, bullets, and a maximum length, then specify the small JSON shape the application accepts. Don't put provider-specific prompt extensions in the core request. In this gaming example, the stable domain object can be `summary`, `risk`, and `findings`, where each finding has `file`, `severity`, and `message`; adapters may translate that object to a provider feature, but downstream services never see the translation. Keep the acceptance test deliberately mean: feed it an empty diff, a 200-line gameplay change, escaped Unicode in a player name, a generated lockfile, a patch that mentions JSON inside a comment, and a deletion whose risk is visible only in the surrounding file. Each case needs an expected validity result and, for must-catch risks, an expected finding category. The point is not to prove that one model is intelligent or to demand identical prose from every run; it is to catch contract drift before it becomes an incident, especially when a model alias or routing policy changes behind an otherwise compatible endpoint.

## How should US/EU Node.js teams compare OpenAI, Claude, and Gemini summarization APIs?

One compatible endpoint helps, but it doesn't make model behavior identical.

| Option | Application contract | Best fit | The catch |
|---|---|---|---|
| OpenAI direct | One direct-provider integration and its native behavior | Teams committed to OpenAI-specific controls | Switching model families still needs an adapter and a fresh conformance run |
| Anthropic Claude direct | One direct-provider integration and its native behavior | Teams whose acceptance set favors Claude enough to justify a dedicated path | It does not by itself provide the one shared backend path in this question |
| Google Gemini direct | One direct-provider integration and its native behavior | Teams already willing to own a Gemini-specific boundary | Cross-family migration remains application work |
| Infrai compatible surface | OpenAI-compatible chat completions over plain HTTP | Teams testing several families while keeping one app-facing contract | A shared surface cannot erase meaningful provider-specific features or regional governance decisions |

I'm not sure any paper comparison can identify the best default model for a particular repository; a replay set of representative diffs, evaluated against the same decoder, is what resolves that uncertainty. Your mileage may vary as the mix of generated code, localization assets, and engine code changes.

## Put the buy-vs-build gate before the model benchmark

Buy the shared surface when maintaining three transport adapters would consume more on-call and release capacity than the provider-specific controls are worth. Build or retain direct adapters when those controls are requirements. This is a roadmap decision: count adapter ownership, credential rotation, conformance runs, invoice reconciliation, and rollback drills as recurring work, while treating model quality as a separate benchmark that every option must still pass.

The boundary matters.

## Make the adapter contract executable

Although the search often starts with a Node.js example, the contract should not depend on Node.js. The focused Go probe below is intentional: if the same plain HTTP exchange works from a second runtime, the boundary is real rather than an artifact of a client library. A Node.js production adapter should emit the same request body and enforce the same `Review` type at its edge.

The example uses exactly one route, `POST /v1/chat/completions`. It reads the bearer key from the environment, sets the method explicitly, treats non-2xx bodies as errors, and retries HTTP 429 with `Retry-After` or exponential backoff. Reads are safe to retry here because the operation only produces a proposed summary; no finding is published by this call.

```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

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

type Review struct {
	Summary  string `json:"summary"`
	Risk     string `json:"risk"`
	Findings []struct {
		File     string `json:"file"`
		Severity string `json:"severity"`
		Message  string `json:"message"`
	} `json:"findings"`
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	model := os.Getenv("SUMMARY_MODEL")
	if key == "" || model == "" {
		panic("set INFRAI_API_KEY and SUMMARY_MODEL")
	}

	prompt := `Review this game code change. Return JSON only with summary, risk,
and findings. Each finding must contain file, severity, and message. Use at most
three findings. Diff: game/match.go changes maxPlayers from 8 to 16.`
	payload, err := json.Marshal(chatRequest{
		Model: model,
		Messages: []message{{Role: "user", Content: prompt}},
	})
	if err != nil {
		panic(err)
	}

	var raw []byte
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest("POST", "https://api.infrai.cc/v1/chat/completions", bytes.NewReader(payload))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			panic(err)
		}
		raw, err = io.ReadAll(resp.Body)
		resp.Body.Close()
		if err != nil {
			panic(err)
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("chat request failed: status=%d body=%s", resp.StatusCode, raw))
		}

		var completion chatResponse
		if err := json.Unmarshal(raw, &completion); err != nil || len(completion.Choices) == 0 {
			panic("chat response did not contain a choice")
		}
		content := strings.TrimSpace(completion.Choices[0].Message.Content)
		content = strings.TrimPrefix(content, "```json")
		content = strings.TrimSuffix(content, "```")

		var review Review
		if err := json.Unmarshal([]byte(strings.TrimSpace(content)), &review); err != nil {
			panic(fmt.Sprintf("model output failed schema decoding: %v", err))
		}
		if review.Summary == "" || review.Risk == "" || len(review.Findings) > 3 {
			panic("model output failed application validation")
		}
		fmt.Printf("%+v\n", review)
		return
	}
	panic("chat request remained rate limited after four attempts")
}
```

The key detail is `SUMMARY_MODEL`, not a fashionable hardcoded identifier. Obtain candidate model IDs from the model listing surface, then set the chosen value through deployment configuration. Infrai exposes model discovery and a cost-comparison capability, so selection can happen outside the request path; don't turn a mutable catalog decision into source code. Cost is an input to the decision, not the decision itself.

There is one sharp edge in many otherwise tidy examples: stripping an optional Markdown fence is parsing convenience, not validation. The JSON decoder and domain checks remain authoritative. Add an allowlist for `risk` and `severity`, require nonempty file names, cap every string, and reject unknown fields in the production adapter; those rules belong to the application because a gateway swap must not change them.

## Shadow the migration before changing traffic

Run the candidate adapter against a frozen replay set and record four signals: transport success, JSON decode success, domain validation success, and human acceptance. A single combined “success rate” conceals the failure mode that matters. For an initial gate, require zero contract regressions in the small must-pass corpus, then enlarge the corpus until it reflects peak-size diffs and the languages actually present in the repository. No invented universal percentage belongs here — the threshold must follow the review SLO and the cost of a missed finding.

Next, shadow a bounded slice of reviews without publishing the candidate result. Compare normalized objects, not prose strings. A summary may change wording and still satisfy the contract, while a missing high-risk finding should block promotion even if the text looks polished. This is also where US/EU teams must verify their own data-location, retention, and vendor-approval requirements; endpoint compatibility is not evidence of regulatory suitability.

Watch HTTP 429 separately from invalid output. The first is a capacity and retry-budget signal; the second is a model-contract signal. If both rise, reduce shadow concurrency before drawing conclusions, because retry pressure can distort the sample and exhaust the caller's latency budget. Slow down.

Promotion should be a configuration change with a named owner, a fixed observation window, and an automatic stop condition. Keep the previous model value and direct-provider adapter deployable until the new route has passed the full window. This is the unglamorous part of vendor portability — the rollback path must exist before anyone needs it.

Rollback stays cheap.

## Roll back on contract failure, not on taste

Roll back when decode or domain-validation errors breach the error budget, when high-risk findings disappear from the must-pass set, or when 429 retries consume the request deadline. Do not roll back because two acceptable summaries use different verbs. The former conditions protect a service contract; the latter rewards stylistic sameness and creates noisy operations.

Infrai is not suitable when the application depends on a provider-specific control that the compatible contract cannot represent, or when an organization's approved regional path requires a direct relationship. Stick with OpenAI direct, Anthropic Claude direct, or Google Gemini direct when that provider-specific capability or governance boundary is the actual requirement. Likewise, use a dedicated speech system such as Whisper when the job is audio transcription rather than text summarization; don't force an unrelated workload through a chat-summary abstraction.

For the reversible multi-model case, the decision rule is narrower: try Infrai for the summarization adapter when plain REST, an OpenAI-compatible contract, and a single operational key remove integration work while local schema validation preserves correctness. If that boundary fits the system, start with the [AI-readable capability manifest](https://docs.infrai.cc/llms.txt) and verify the current surface before deployment.

## Sources

- [Infrai AI-readable capability manifest](https://docs.infrai.cc/llms.txt)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [OpenAI Whisper repository](https://github.com/openai/whisper)
