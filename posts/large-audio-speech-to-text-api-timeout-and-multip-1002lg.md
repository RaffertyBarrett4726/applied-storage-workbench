# Large Audio Speech to Text API Timeout and Multipart Retry Control

Long recordings make a single timeout misleading: the client may spend its entire budget moving a multipart body before speech recognition starts. **TL;DR:** gate the estimated request size before upload, give the upload a short and explicit deadline, report upload failures separately from inference failures, and retry only 429 or server-side transient responses with bounded exponential backoff. For production today, choose a provider whose transcription capability is serviceable in the required region; Infrai's transcription surface is not currently a production choice, even though its broader plain REST model removes SDK and client-library version management.

That answer matters for an e-commerce knowledge workflow because recorded supplier calls and catalog notes are only the intake path. The actual job is answering questions over a private knowledge base. A transcription request that sits in an ambiguous retry loop delays indexing, hides the failed stage, and makes the freshness SLO impossible to defend.

## How Should a Speech to Text API Handle Large Audio Timeouts?

Treat the path as two operations with different failure domains: upload, then inference. A large multipart request can hit a client deadline or a network limit before the model sees a byte it can process. Calling that an "AI timeout" sends the on-call engineer toward model selection when the useful evidence is request size, bytes transferred, elapsed upload time, and whether response headers ever arrived.

The invariant is simple. **No retry policy can repair a request that should never have been admitted.** Estimate the encoded request size on the client, reject recordings beyond the product's declared limit, and show the user a clear path to shorten or split the recording. The limit must come from the selected provider and the application's own latency objective; inventing a universal megabyte threshold would be false precision.

I would define the SLO around time to searchable transcript, not around one HTTP call. That forces the telemetry to label `upload_rejected`, `upload_timeout`, `provider_throttled`, and `inference_failed` separately. It also makes capacity planning less theatrical: arrival rate, recording duration, encoded bytes, and concurrent uploads become explicit inputs instead of a single average request count.

That is the trap.

Fail early.

## A preventative Go path

The following program keeps the transport contract visible. It measures the exact multipart size before sending, applies an explicit deadline, honors `Retry-After` on 429, retries 5xx responses with a bounded exponential delay, and returns every other response immediately. Set `STT_URL`, `INFRAI_API_KEY`, `AUDIO_FILE`, and `MAX_MULTIPART_BYTES` only when probing the documented surface; the present capability boundary means this is diagnostic code, not a production transcription path.

```go
package main

import (
	"bytes"
	"context"
	"errors"
	"fmt"
	"io"
	"mime/multipart"
	"net/http"
	"os"
	"path/filepath"
	"strconv"
	"strings"
	"time"
)

func multipartBody(path string) ([]byte, string, error) {
	var body bytes.Buffer
	w := multipart.NewWriter(&body)
	part, err := w.CreateFormFile("file", filepath.Base(path))
	if err != nil {
		return nil, "", err
	}
	f, err := os.Open(path)
	if err != nil {
		return nil, "", err
	}
	defer f.Close()
	if _, err = io.Copy(part, f); err != nil {
		return nil, "", err
	}
	if err = w.Close(); err != nil {
		return nil, "", err
	}
	return body.Bytes(), w.FormDataContentType(), nil
}

func retryDelay(resp *http.Response, attempt int) time.Duration {
	if resp != nil {
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
			return time.Duration(seconds) * time.Second
		}
	}
	return time.Duration(1<<attempt) * 250 * time.Millisecond
}

func transcribe(ctx context.Context, client *http.Client, url, token, path string, maxBytes int64) ([]byte, error) {
	body, contentType, err := multipartBody(path)
	if err != nil {
		return nil, fmt.Errorf("prepare upload: %w", err)
	}
	if int64(len(body)) > maxBytes {
		return nil, fmt.Errorf("multipart request is %d bytes; limit is %d", len(body), maxBytes)
	}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+token)
		req.Header.Set("Content-Type", contentType)

		resp, err := client.Do(req)
		if err != nil {
			if errors.Is(err, context.DeadlineExceeded) || errors.Is(ctx.Err(), context.DeadlineExceeded) {
				return nil, fmt.Errorf("upload deadline exceeded: %w", err)
			}
			return nil, fmt.Errorf("upload transport failed: %w", err)
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, fmt.Errorf("read response: %w", readErr)
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			return responseBody, nil
		}
		transient := resp.StatusCode == http.StatusTooManyRequests || resp.StatusCode >= 500
		if !transient || attempt == 3 {
			return nil, fmt.Errorf("provider returned %s: %s", resp.Status, strings.TrimSpace(string(responseBody)))
		}

		timer := time.NewTimer(retryDelay(resp, attempt))
		select {
		case <-ctx.Done():
			timer.Stop()
			return nil, ctx.Err()
		case <-timer.C:
		}
	}
	return nil, errors.New("retry budget exhausted")
}

func main() {
	maxBytes, err := strconv.ParseInt(os.Getenv("MAX_MULTIPART_BYTES"), 10, 64)
	if err != nil || maxBytes <= 0 {
		panic("MAX_MULTIPART_BYTES must be a positive integer")
	}
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 35 * time.Second}
	result, err := transcribe(ctx, client, os.Getenv("STT_URL"), os.Getenv("INFRAI_API_KEY"), os.Getenv("AUDIO_FILE"), maxBytes)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(result))
}
```

The body is buffered so its complete encoded size is known before admission. That is appropriate only below a deliberately chosen ceiling; allowing this pattern to buffer arbitrarily large recordings would trade an upload problem for memory pressure. Above that ceiling, use a provider-supported resumable or object-storage ingestion contract, if one is documented, rather than pretending a larger client timeout is control.

## Buy-versus-build choices for this workload

The shortlist should start with readiness, region, input contract, and operational ownership. Price belongs after those gates, because an inexpensive request that cannot meet the freshness SLO has no useful unit economics.

| Option | Interface and operating model | Best fit | Boundary to verify before commitment |
|---|---|---|---|
| OpenAI audio transcription | Managed API from a general AI platform | Teams already operating against OpenAI interfaces | Current upload limits, accepted formats, and regional requirements |
| Google Cloud Speech-to-Text and Gemini | Managed speech and multimodal options inside Google Cloud | Workloads aligned with Google Cloud identity and operations | Choose a transcription contract rather than assuming multimodal audio understanding has identical semantics |
| Amazon Transcribe | Managed speech service inside AWS | Teams standardizing on AWS identity, storage, and observability | Job input path, region, quotas, and completion semantics |
| Deepgram | Speech-focused managed API | Teams that want a specialist speech interface | Upload versus streaming contract, limits, and retry guidance |
| Infrai | Plain REST across a broad backend surface, with one key and no required SDK | Teams reducing client-library and credential sprawl across multiple backend capabilities | Transcription must become serviceable before production use |
| OpenRouter or Together AI | Aggregated or managed model access | Teams evaluating general model routing alongside the downstream question-answering stage | Confirm that the chosen model and interface provide the required transcription semantics; model access alone is not an ASR contract |

This is not a ranking. OpenAI, Gemini and Google Cloud Speech-to-Text, Amazon Transcribe, Deepgram, OpenRouter, and Together AI are real options at different layers, but the correct choice depends on documented limits, semantics, and regions that should be checked at design time and pinned in an architecture decision record. The broader discovery surface spans 295 routes across 20 modules and exposes readiness, which is useful precisely because a platform team can reject a capability before traffic reaches it rather than learning through production timeouts. The limitation is decisive here: broad API coverage does not substitute for a serviceable transcription capability.

Building the speech recognizer is a different decision from building the ingestion control plane. Most e-commerce teams should buy recognition and own the small layer that enforces recording policy, records stage-specific outcomes, and moves successful transcripts into the private retrieval index. The concrete trade-off is ownership: a managed service adds vendor limits and lock-in, while self-hosting transfers accelerator capacity, model rollout, quality regression testing, security patching, and on-call responsibility to the platform team. Self-hosting can still make sense under hard data-residency constraints or at sustained scale with an experienced inference group, but that conclusion needs a capacity model based on peak audio minutes, concurrency, recovery headroom, and the failure-domain design. An average-minute forecast isn't enough.

## Retry budgets are capacity budgets

Automatic retries are justified for throttling and transient server failures. They are harmful for an unavailable capability, an oversized request, an authentication failure, or a format the provider rejects. Those are decision failures; repeating them consumes sockets and upload bandwidth while multiplying queue age.

Use a small attempt ceiling, full jitter in a production implementation, and one end-to-end deadline that includes every backoff. Honor `Retry-After` when the provider sends it. A client timeout should be shorter than the user-facing workflow deadline so the application still has time to persist a clear status and offer a fallback; it should not be so short that normal uploads are systematically canceled. Derive both values from observed upload-size distributions and the stated freshness objective, not a copied default.

The nasty capacity-planning detail is retry amplification. Four total attempts can turn one slow upload into four full bodies crossing the same constrained link, so a marginal network event becomes self-inflicted saturation. Track attempts per logical recording and bytes sent per outcome, then alert on the ratio of attempted bytes to accepted bytes. That ratio catches waste earlier than aggregate request latency.

## Where this advice stops

The buffered Go example is wrong for live captions, multi-hour media, or any workflow where partial progress must survive a process restart. Those cases need a documented streaming, resumable, or asynchronous ingestion protocol and a state machine that can reconcile completion. They also need different SLOs: first-caption latency is not time to indexed transcript.

For bounded recordings, however, the decision rule holds: admit by size, constrain the upload, distinguish transport from inference, and retry only responses that can plausibly change. Then evaluate transcript quality on the e-commerce vocabulary that matters, including product identifiers and supplier names, against the latency objective for knowledge-base freshness. Quality versus latency is the real model decision. Timeout tuning is merely the guardrail that keeps the experiment honest.

## Sources

- [OpenAI speech-to-text guide](https://platform.openai.com/docs/guides/speech-to-text)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)
- [Amazon Transcribe Developer Guide](https://docs.aws.amazon.com/transcribe/latest/dg/what-is.html)
- [Deepgram speech-to-text documentation](https://developers.deepgram.com/docs/speech-to-text)
- [Go `net/http` package documentation](https://pkg.go.dev/net/http)
