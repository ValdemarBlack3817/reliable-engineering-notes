# A 4-Signal Email Deliverability Fallback Strategy for SMS Alerts

Short answer: poll the email provider's event feed into a durable local state machine, and send a text only after a terminal bounce for the current signup attempt. Do not make a missing email event equivalent to a bounce. The least complex defensible design uses one worker, one database table, an idempotency key at the SMS boundary, and four signals: event-ingestion lag, terminal-bounce age, verification completion, and text-attempt outcome.

The page says `signup_verification_stalled`, not `email_provider_down`. On-call sees the affected region, the oldest unresolved signup age, the event-feed cursor age, and counts split by channel state. The immediate action is to check whether polling is fresh; only then should anyone consider changing a threshold or replaying work. An aggressive fallback can turn a delayed event feed into duplicate messages, unnecessary consent exposure, and a second provider incident.

## How should an email deliverability fallback strategy trigger an SMS alert?

The earlier signal is ingestion lag. If the poller has not advanced its provider cursor, the system does not know whether a message bounced, remains in flight, or produced an event the local worker has not observed. Alerting first on stalled signups confuses customer impact with incomplete evidence. Page on the evidence pipeline before its uncertainty consumes the recovery budget.

Silence proves nothing.

Set the budget from the signup SLO rather than a vendor's default retry interval. For an illustrative 15-minute verification objective, a team might reserve 5 minutes for email delivery and bounce evidence, 5 minutes for text recovery, and 5 minutes for the user to act. Those are planning inputs, not universal facts. Measure the actual event delay and completion distribution in each operating region before choosing them. US and EU traffic should have separate dashboard dimensions when routing, consent rules, or providers differ; geography is a diagnostic label, never a reason to infer consent.

A page needs four timestamps: signup creation, last successful event poll, terminal bounce observation, and fallback attempt. Without them, on-call cannot distinguish a slow mailbox from a stuck poller.

The trap is subtle. A bounce counter can look healthy while the event feed is frozen, because zero newly observed bounces is also what a perfect delivery interval looks like. A freshness gauge closes that blind spot.

This polling strategy has a real limitation: it is a poor fit when the source cannot provide stable event identifiers, ordered pagination, or a documented retention window long enough to recover from an outage. In that case, a managed push subscription with replay, or a transactional outbox under the team's control, may offer a clearer evidence boundary. Push delivery still needs deduplication and replay handling, so swapping polling for a webhook does not remove the state machine; it changes who schedules delivery and which freshness signal pages. Polling also trades lower inbound exposure for repeated reads and detection delay. Teams with tight recovery budgets must prove that the feed's rate limits and worst-case page depth allow the backlog to drain before adopting it.

That trade-off is nontrivial.

## Model evidence before actions

Treat provider events as observations, not commands. Normalize each event into a small internal vocabulary, retain the provider event identifier for deduplication, and attach it to one immutable signup attempt. A later signup for the same address is a new attempt; an old bounce must never trigger its recovery.

The transition requires a matching active attempt, a terminal bounce rather than silence, an account that remains unverified, established text consent and destination eligibility, and no text attempt for the same recovery key.

SMTP distinguishes permanent failures, represented by a `5yz` reply, from transient failures, represented by `4yz`; RFC 5321 says a client generally should not repeat a command after a permanent negative completion reply. Enhanced status codes add machine-readable detail, but local policy still needs an explicit mapping because event payloads are provider boundaries. Do not escalate on opens, delayed delivery, a polling timeout, or an unknown event. Quarantine unknown values and alert on the mapping gap.

| Concern | Managed boundary | Platform-owned boundary | SLO consequence |
|---|---|---|---|
| Mailbox evidence | Provider event feed | Cursor, normalization, deduplication | Stale evidence pages before recovery stalls |
| Recovery decision | None | Consent, correlation, state transition | Duplicate decisions spend error budget |
| Text transport | Provider submission | Idempotency, retry, outcome record | Ambiguous submission blocks blind retry |
| User completion | Verification endpoint | Token and completion state | Completion cancels queued recovery |

Buying transport reduces protocol and carrier work, but it does not transfer ownership of correlation, consent, or the user-facing SLO. Building the poller keeps decision logic portable, while adding cursor storage, schema-drift handling, regional capacity planning, and an on-call surface. That is the buy-versus-build trade, not the number of API calls in a quick-start guide.

## Instrument the polling boundary

Advance a durable cursor only after the fetched page has been committed. Process events idempotently because a crash between commit and acknowledgement can replay a page, and pagination contracts can expose overlaps. Use a unique constraint on the provider event ID plus a recovery key such as `signup_attempt_id + channel + purpose`.

This Go example focuses on the decision boundary. Its interfaces can wrap HTTP clients in a Node.js service; the state invariants matter more than the runtime.

```go
package recovery

import (
    "context"
    "time"
)

type Event struct {
    ID, AttemptID, Kind string
    Occurred            time.Time
}

type EventSource interface {
    Poll(context.Context, string, int) ([]Event, string, error)
}

type Repository interface {
    RecordEvent(context.Context, Event) (bool, error)
    EligibleForText(context.Context, string) (bool, error)
    ClaimText(context.Context, string) (bool, error)
    CommitCursor(context.Context, string) error
}

type TextSender interface {
    SendVerification(context.Context, string, string) error
}

type Worker struct {
    Events EventSource
    Repo   Repository
    Texts  TextSender
}

func (w Worker) PollOnce(ctx context.Context, cursor string) (string, error) {
    events, next, err := w.Events.Poll(ctx, cursor, 100)
    if err != nil {
        return cursor, err
    }
    for _, event := range events {
        inserted, err := w.Repo.RecordEvent(ctx, event)
        if err != nil {
            return cursor, err
        }
        if !inserted || event.Kind != "terminal_bounce" {
            continue
        }
        eligible, err := w.Repo.EligibleForText(ctx, event.AttemptID)
        if err != nil || !eligible {
            if err != nil {
                return cursor, err
            }
            continue
        }
        key := "signup-verification:" + event.AttemptID
        claimed, err := w.Repo.ClaimText(ctx, key)
        if err != nil {
            return cursor, err
        }
        if claimed {
            if err := w.Texts.SendVerification(ctx, event.AttemptID, key); err != nil {
                return cursor, err
            }
        }
    }
    if err := w.Repo.CommitCursor(ctx, next); err != nil {
        return cursor, err
    }
    return next, nil
}
```

Export `last_successful_poll_time` and `last_cursor_advance_time` separately. A healthy empty poll updates the first, while only observed forward progress updates the second. If a cursor changes only when events exist, paging on cursor age manufactures noise; use successful-poll age plus a synthetic event instead.

Capacity planning starts with bursts. Size the worker for the largest plausible backlog that can drain inside the evidence budget, account for rate limits, and cap concurrency so recovery traffic cannot starve ordinary verification sends. Queue depth, oldest event age, fetch latency, normalization failures, and duplicates belong on one view.

## Test the recovery clock

Freeze time in unit tests and prove that a transient failure does not trigger a text, a terminal bounce does, and replaying an event produces one claim. Integration tests should crash after recording an event but before committing its cursor. The next run must replay safely. Also test verification completing after eligibility is read but before the text claim; a transactional claim that rechecks completion prevents the stale action.

Run a synthetic signup through each regional route using controlled destinations and explicit consent. It should verify submission, event retrieval, normalization, and completion without generating customer-facing text every run. A separate, lower-frequency controlled exercise can cover fallback. Label synthetic data so it cannot contaminate deliverability reporting.

Deploy in shadow mode first: record `would_send_text` decisions without sending, then compare them with eventual completions, late delivery, and terminal bounce evidence. The observation period must reflect the measured delay distribution. Roll out by region and retain a kill switch that stops new text claims without stopping event ingestion. Evidence must keep flowing during mitigation.

Keep polling.

## Where should the threshold land?

Place it early enough that on-call can restore polling and preserve the recovery budget, but late enough to ignore normal scheduling jitter. A multi-window burn-rate alert is generally more useful than a fixed count because it can detect a sharp regional failure and a slower budget leak while tying urgency to the signup SLO. Exact windows and burn rates depend on the stated SLO and traffic volume.

A one-minute freshness page for a five-minute polling schedule is guaranteed noise. An aggregate threshold can hide a low-volume region. A bounce-rate page with no sample floor or freshness guard can fire on one event, waking an engineer without producing a useful action.

False positives consume on-call capacity through interruptions, desensitization, and risky manual intervention. They also affect customers if a runbook tells responders to force text delivery while email status is uncertain. Track pages that led to no corrective action, but do not optimize that count to zero. Review whether each page arrived with enough state for a safe choice. **Page on lost evidence, automate text only on correlated terminal evidence, and stop recovery when verification succeeds.**

## Further reading

- RFC 5321, Simple Mail Transfer Protocol: https://www.rfc-editor.org/rfc/rfc5321
- RFC 3463, Enhanced Mail System Status Codes: https://www.rfc-editor.org/rfc/rfc3463
- Google SRE Workbook, Alerting on SLOs: https://sre.google/workbook/alerting-on-slos/
- OWASP Authentication Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- NIST Digital Identity Guidelines: https://pages.nist.gov/800-63-4/sp800-63b.html
