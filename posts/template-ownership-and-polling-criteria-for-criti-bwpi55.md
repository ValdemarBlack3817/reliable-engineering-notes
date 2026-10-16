# Template Ownership and Polling Criteria for Critical SMS API Outage Alerts

The critical outage alert says that settled payments are no longer producing patient order receipts, but the on-call cannot tell whether the fault is rendering, SMS API submission, carrier acceptance, or the final handset hop. Choose an API that lets the application own versioned templates and recipient policy while exposing a stable message identifier, monotonic delivery states, timestamps, and explicit retry boundaries in both the US and EU.

**TL;DR:** choose from evidence gathered in a failure drill, not from a feature matrix. Submit one synthetic receipt, retain its provider-neutral correlation ID, poll until a terminal state, inject a timeout, suppress automatic duplicates, and verify that cancellation has a precisely documented boundary. Template ownership matters because an outage should not force responders to edit copy in an external console while they are also deciding whether a patient has already received a legally or clinically sensitive message.

## How should teams choose an SMS API for critical outage alerts?

The page is late.

It reports the business consequence after the queue has already accumulated settled orders whose receipts have no terminal outcome. A useful earlier signal is the age of the oldest unsettled receipt delivery record, split by destination region and lifecycle state; a raw count of failed API calls misses accepted messages that later stall, while a provider-wide health alarm says little about this particular workflow. The practical trade-off is sensitivity against responder attention: a faster threshold can expose a stalled regional path earlier, but normal tail latency can then wake someone who has no action to take.

Set an SLO around the outcome the system controls: every eligible settled order must acquire a terminal receipt state within the workflow's stated time objective, excluding recipients that policy marks ineligible before submission. Do not quietly translate `accepted` into `delivered`. Acceptance proves that one boundary took custody. It does not prove handset delivery.

For the page, include the order correlation ID, template version, destination region, current state, state age, attempt count, and last transition time. Exclude message bodies and full phone numbers. The responder needs enough evidence to choose an action without putting patient or payment context into a paging system.

## Trace one alert backward

Start at the action the responder can take. If a receipt has no provider message ID, the failure is before or during submission; retrying may be valid only when the application can reuse an idempotency key or reconcile an ambiguous response. If it has an ID and a nonterminal state, poll within a bounded window. If it is terminally failed, create a new attempt only after policy says a duplicate is preferable to silence. If it is delivered, stop. Never resend merely because a poll timed out.

Cancellation is narrower than it sounds.

Treat it as a request that may succeed only before an irreversible downstream transition, and require the API to return an auditable result rather than letting the application pretend that deleting its local job recalled a carrier message. This design is not suitable when the workflow requires guaranteed recall after carrier acceptance; SMS cannot be treated as a reversible database transaction, so that requirement calls for a different notification channel or a message containing no sensitive content.

The state machine below keeps the vendor adapter small. The particular transport can map its vocabulary into these states, but it cannot change their meaning.

```go
package receipt

import (
	"context"
	"errors"
	"time"
)

type State string

const (
	Queued    State = "queued"
	Accepted  State = "accepted"
	Delivered State = "delivered"
	Failed    State = "failed"
	Canceled  State = "canceled"
)

type Status struct {
	MessageID  string
	State      State
	ObservedAt time.Time
}

type Gateway interface {
	Send(ctx context.Context, idempotencyKey, templateVersion, destination string) (Status, error)
	Status(ctx context.Context, messageID string) (Status, error)
	Cancel(ctx context.Context, messageID string) (Status, error)
}

func Terminal(s State) bool {
	return s == Delivered || s == Failed || s == Canceled
}

func Wait(ctx context.Context, g Gateway, id string, interval time.Duration) (Status, error) {
	ticker := time.NewTicker(interval)
	defer ticker.Stop()

	for {
		status, err := g.Status(ctx, id)
		if err == nil && Terminal(status.State) {
			return status, nil
		}
		select {
		case <-ctx.Done():
			return Status{}, errors.New("delivery status deadline exceeded")
		case <-ticker.C:
		}
	}
}
```

This example deliberately does not resend. A caller must reconcile the old attempt, evaluate consent and duplicate risk, then issue a new `Send` with a new attempt record. That separation is inconvenient. It is also the point: retries repair an uncertain operation; resends create another message.

## Instrument the boundary, then test it

Record each transition as an append-only event with the application correlation ID, provider message ID when known, attempt number, normalized state, region, template version, and observed timestamp. Derive queue age and terminal latency from those events. Counters alone cannot answer the incident question, “Did this order already receive a receipt?”

The drill should cover at least four outcomes: submission rejected before an ID exists, submission timing out with an ambiguous result, acceptance that remains nonterminal beyond the objective, and terminal failure followed by a policy-approved resend. Add an out-of-order status update. The normalizer must reject regressions such as `delivered` returning to `accepted`, while retaining the raw event for audit.

Capacity planning starts with polling amplification. At 30,000 unsettled receipts and a 10-second interval, the system asks for 3,000 status reads per second before retries, responder queries, or regional failover. That number is arithmetic, not a benchmark or a proposed limit. Measure the candidate API's documented quotas and observed throttling in a test account, then choose polling cadence, jitter, and concurrency against the worst credible backlog. Push callbacks may reduce polling load, but they do not remove the need for reconciliation after missed or delayed callbacks.

Instrument one histogram for time from payment settlement to terminal delivery state, and gauges for oldest nonterminal age and records missing a provider ID. Page on sustained objective risk, not on a single transient request error. Ticket slower template-rendering regressions separately so the outage channel retains meaning.

## Template ownership is an operational boundary

Application-owned templates give code review, version pinning, deterministic test fixtures, and one deployment record for the receipt workflow. Provider-owned templates can centralize controls outside the application and may be required by a transport or jurisdictional process, but they add another mutable production surface. The selection question is not “Does the API support templates?” It is “Who can change the exact message that version 42 means during an incident, and how is that change audited and rolled back?”

For a healthtech receipt, keep medical detail out of SMS and make the template data contract explicit. A template should accept the minimum fields needed to identify the order receipt safely; the renderer should reject unknown template versions rather than silently falling back to whatever is current in a console.

| Decision | Managed template control | Application template control | Self-hosted transport layer |
|---|---|---|---|
| On-call load | Lower routine operation, extra control plane during incidents | One release path, application team owns rendering failures | Team owns rendering, routing, carrier integration, and escalation |
| Audit boundary | Provider history plus application records | Repository and deployment history | Entire stack and its retention policy |
| Lock-in pressure | Template identifiers and console workflow | Adapter mainly maps delivery states | Carrier contracts and protocol operations move in-house |
| Best evidence to request | Version history, rollback behavior, regional availability | Payload limits, idempotency, lifecycle semantics | Staffing, routing expertise, compliance review, capacity tests |

No row decides the purchase.

A small team may rationally buy operational coverage while retaining template source in its repository; a larger platform team may accept more ownership to keep message policy portable. Application ownership has a real limitation: the team becomes responsible for rendering correctness, deployment coordination, and every template migration. Provider ownership is a poor fit when reproducible builds and repository review are mandatory, while self-hosting is a poor fit when the on-call rotation lacks carrier-routing expertise. Price belongs in the total operating model alongside responder load, regional routing work, evidence retention, and migration cost, not as the headline criterion.

## The selection exercise

Give every candidate the same test harness and the same fixture: a settled order, a pinned receipt template, a pseudonymous destination, and a correlation ID generated by the application. Score only observable behavior. Can the team reconcile an ambiguous submission? Are state meanings and terminal states documented? Can polling be bounded without losing evidence? Does a cancellation response distinguish “stopped” from “too late”? Can template changes be reviewed, pinned, audited, and rolled back? Are US and EU data paths, retention controls, and support escalation boundaries clear enough for the organization's own review?

Require artifacts: event logs from the drill, the normalized state mapping, quota documentation, a template-change audit, and an incident escalation path. A slide saying “global delivery” is not evidence.

Use the same adapter contract during evaluation, even if the production application is written in another language. Tool interfaces work best when names, inputs, and outcomes are explicit; the same discipline prevents an automated responder or human runbook from confusing `retry`, `resend`, and `cancel`. The application should authorize each action, validate its arguments, and store the result.

## Thresholds have a false-positive budget

An oldest-message page set too close to normal tail latency trains responders to distrust it and can provoke duplicate receipts. Set it too far out and the business-impact page becomes the first useful signal again. Establish the threshold from the stated delivery objective and measured latency distribution, require a sustained breach or a corroborating backlog signal, and review it after volume or routing changes.

The final decision should preserve one property above all: during an incident, the responder can prove what copy was selected, whether the transport accepted it, what happened next, and whether another send is warranted. That is a stronger selection criterion than an SDK demo, and it remains useful when carriers, regions, or application languages change.

## Further reading

- RFC 6376, DomainKeys Identified Mail (DKIM): https://datatracker.ietf.org/doc/html/rfc6376
- Anthropic, Tool use overview: https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
