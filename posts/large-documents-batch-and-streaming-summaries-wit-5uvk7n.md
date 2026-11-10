# Large Documents: Batch and Streaming Summaries with Portable CRM Release Gates

Use queued batch execution for large sales-call documents whose CRM actions can wait, and stream a review preview only when a person needs to read before processing finishes. TL;DR: choose the completion deadline first, keep generated text provisional, and release CRM writes through a separate validation and approval gate. A portable implementation must preserve that gate when the model provider changes.

For a developer-tools platform summarizing sales calls into CRM actions, the useful operational distinction is between a readable draft and a completed, approved action. Streaming can serve the first milestone; batch execution can serve a backlog with an agreed deadline. Neither delivery mode should determine whether a proposed customer commitment becomes a CRM record.

This runbook uses a proposed workload of 120 call transcripts arriving within 30 minutes, with a two-hour deadline for reviewed action proposals. These are planning inputs, not measured throughput or a service guarantee. Replace them with the team's actual arrival curve before selecting capacity.

## Should batch or streaming summarize large documents before CRM review?

Start with two separate SLOs: time until a reviewer can see useful draft text, and time until an approved CRM action is durably recorded. Define the first around the review workflow, because an early sentence that cannot yet support a decision is a weak success criterion; define the second around the complete pipeline, including queue time, summarization, evidence validation, approval, and the destination write. Treat any numerical targets as local policy. For the example workload, asynchronous batch execution is a reasonable default if the two-hour deadline survives queueing and review. Here, batch means queued work with tracked completion, rather than a claim about any provider's bulk endpoint. A streaming request is an optional presentation path for a waiting reviewer. Require both paths to produce the same final proposal format before downstream processing.

The trade-off is explicit: batch execution is not suitable for a workflow that requires a reviewer to inspect a draft before the batch completes. Choose a streaming preview for that requirement, accepting the additional state needed to distinguish provisional output from a completed proposal. Streaming has a limitation too: a preview cannot satisfy an SLO defined as a confirmed CRM write. Don't substitute its first-fragment latency for that completion target.

Count the backlog.

If the system cannot finish the queued work before its deadline, exposing partial text does not repair the completion budget. Reserve time for review and recovery before allocating the remainder to model execution, and verify that a substitute provider can meet that budget using its own measured service times and concurrency constraints. Do not infer completion capacity from how quickly the first fragment appears.

## Diagnose the backlog before changing the delivery mode

Instrument receipt, enqueue, dispatch, first useful preview, final response, validation, approval, and confirmed CRM write. Keep stage timestamps under a stable local job identifier. Provider request identifiers can accompany that record, but a migration should not require them to reconstruct what the application owes a reviewer.

The signal that warrants intervention is a shrinking deadline margin: the oldest unfinished job approaches the point where the remaining execution and review work cannot fit. In the proposed burst, the average arrival rate is four transcripts per minute; that arithmetic says nothing about document lengths, retries, or reviewer availability, so size the test corpus to include long calls and action-heavy discussions before treating that rate as a capacity target. Measure document preparation and review time as well as model time. A fast generation stage is little comfort when review consumes the remaining budget.

A second signal is disagreement between the preview and the final action proposal. For example, a draft may display an apparent follow-up owner before later transcript evidence changes that assignment. The safe response is to keep the preview visibly provisional and require a completed proposal with a source span for each action; do not let the interface convert each incoming text fragment into a destination update.

Hold the write.

For large documents, propose a bounded extraction stage that retains transcript identifiers and source offsets, followed by a consolidation stage that resolves duplicated or conflicting proposed actions. Treat these as application design choices to test, not claims that a particular chunk size preserves every relevant detail. A statement near one document boundary may qualify an action near another. Keep unresolved assignments available for review instead of forcing a confident answer.

## Implement one release gate for both execution paths

Give the application ownership of the job record, source references, proposal version, and write ledger. Put provider-specific submission, completion, and stream handling behind an adapter whose output is a completed proposal. The adapter must explicitly report an interrupted or incomplete result; callers should not have to guess completion from the last fragment they received.

The following Go example is a deliberately narrow release gate. Its inputs describe checks performed elsewhere: source verification, schema validation, and approval of this exact proposal revision. It demonstrates where the decision belongs; it does not perform those checks or implement a durable destination transaction.

```go
package release

import "errors"

type Candidate struct {
    JobID            string
    Revision         string
    Complete         bool
    SchemaValid      bool
    EvidenceVerified bool
    ApprovedRevision string
}

func Check(c Candidate) error {
    if c.JobID == "" || c.Revision == "" {
        return errors.New("missing job or revision")
    }
    if !c.Complete {
        return errors.New("proposal is incomplete")
    }
    if !c.SchemaValid || !c.EvidenceVerified {
        return errors.New("proposal requires validation")
    }
    if c.ApprovedRevision != c.Revision {
        return errors.New("proposal revision requires approval")
    }
    return nil
}
```

Bind approval to the proposal revision because a regenerated summary may change an owner, deadline, or commitment after a person has approved an earlier version. Persist the approved payload as well as its revision. Passing this function authorizes an attempt; the write ledger still needs to distinguish a confirmed destination update from an attempt whose result is unknown. Define and test a destination reconciliation procedure before enabling automatic retries. HTTP method semantics alone do not establish that a particular CRM operation can safely be repeated. Keep the proposal schema deliberately small: action text, source span, proposed owner, proposed due date, and an explicit unresolved state. Use a documented JSON contract and validate it at the application boundary. Syntax validation cannot establish that a customer actually promised something. That requires checking the source, and ambiguous evidence should reach the reviewer.

MDN describes WebRTC facilities for real-time communication. A live call capture path may need such facilities; a completed transcript awaiting an action summary presents a different application requirement. Choose capture transport separately from summary execution, so changing the latter does not imply rebuilding the former.

## Assign the operational work before choosing buy or build

Provider portability is an acceptance constraint for this workflow, while deadline headroom is the day-to-day operating lever. Evaluate each execution option against the same replay corpus, review gate, and destination ledger. The following buy-vs-build table assigns work to inspect during evaluation; it does not assert features or guarantees for an unnamed service.

| Option | Evidence to demand | Work the platform team retains |
| --- | --- | --- |
| Buy managed batch execution | Completion tracking, measured deadline behavior, and documented failure handling | Review policy, source verification, reconciliation, and exit testing |
| Build a queue around request workers | Load tests, retry limits, worker recovery, and backlog visibility | Queue operations, adapter maintenance, and capacity planning |
| Build and operate the inference service | Measured quality and throughput on the same transcript corpus | Serving operations, deployment recovery, and resource planning |

A managed execution path is acceptable only after its completion behavior fits the application deadline and its results can enter the local proposal contract. Owning a queue is acceptable only when someone owns its recovery procedure and on-call alerts. Self-hosting should face the same test, with serving operations included in the responsibility estimate. Compare staff time, review effort, capacity headroom, and migration work alongside spend; a unit price cannot settle those obligations.

The practical migration test is to replace an adapter and replay unfinished jobs without losing approvals or duplicating confirmed CRM actions. If the application needs to reinterpret historical provider events to do that, keep the migration in rehearsal until the job record contains enough local state.

## Verify the deadline, then rehearse rollback

Use a fixed, reviewed corpus containing a long transcript, a call with no action, an ambiguous owner, a corrected commitment, and a repeated follow-up. Execute it through both delivery paths and inspect the final proposals against source evidence. The acceptance criteria should cover supported actions, unresolved ambiguity, review latency, and confirmed destination effects. A polished preview is insufficient evidence.

Exercise interruption during streaming, delayed batch completion, a duplicate completion notification, and a lost response from the CRM write. These are proposed fault-injection cases. Check that incomplete results remain provisional, duplicate delivery does not create another write, and an uncertain write enters reconciliation instead of an unconditional retry. Also verify that approval of one revision cannot authorize a later revision.

Deploy an adapter change to a bounded cohort with destination writes initially held for review. Preserve the prior adapter configuration and retain source documents under the team's retention policy. Set rollback thresholds from the local SLO before release: excessive deadline misses, evidence-validation failures, or unreconciled writes should stop expansion.

Rollback starts by pausing new dispatch to the changed adapter and holding pending writes. Reconcile outstanding jobs, preserve confirmed destination updates, and redispatch only work whose status permits another execution. Rolling back model execution cannot undo an action already written to the CRM; use the destination's established correction procedure when a confirmed record needs repair.

Keep the operational rule simple: queue deadline-tolerant summaries, stream previews when review benefits, and require the same completed, evidence-checked, revision-approved proposal before any CRM action is released. Rehearse that rule across adapters before relying on provider portability during an incident.

## References

- MDN, WebRTC API: https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API
- RFC 8259, The JavaScript Object Notation (JSON) Data Interchange Format: https://www.rfc-editor.org/rfc/rfc8259
- RFC 9110, HTTP Semantics: https://www.rfc-editor.org/rfc/rfc9110
