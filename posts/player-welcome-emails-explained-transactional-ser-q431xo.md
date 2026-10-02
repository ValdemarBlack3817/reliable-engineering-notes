# Player Welcome Emails Explained: Transactional Service Deliverability with SPF and DKIM

TL;DR: Treat an API acceptance as the start of a delivery trace, not proof that a welcome email reached a player. For a gaming contact form, preserve one correlation ID across form intake, queue routing, message submission, provider events, and support handling; verify the sending domain with SPF and DKIM; and page only on symptoms that threaten the delivery SLO. A provider should earn its place through observable state transitions, stable idempotency behavior, and exportable event data. SMTP relay is unnecessary when the application owns an API-first submission path.

Acceptance is not delivery.

The page says `welcome_delivery_gap`: accepted welcome messages have stopped reaching a terminal delivery state for 12 minutes. The on-call sees the affected sending domain, region, template revision, a count of messages stuck after acceptance, and links to redacted traces. At the same time, player contact forms are still entering support queues, so this is not a generic application outage. It is a narrower failure with an ugly consequence: new players can ask for help, but they may never receive the message that tells them their account is ready.

That alert is useful because it names a user-visible gap. A page saying "email API errors increased" is weaker: a retryable submission error may be absorbed by the queue, while a stream of successful `202` responses can hide a broken sender domain or missing downstream events. Capacity planning starts at that distinction. Submission throughput, event-ingestion throughput, and the age of the oldest unresolved message need separate budgets because each resource can saturate alone.

## Can a transactional email service prove welcome email deliverability after acceptance?

The earlier signal is an aging state transition. Every welcome message begins as `queued`, moves to `accepted` only after the delivery service acknowledges it, and becomes `delivered`, `deferred`, `bounced`, or `suppressed` when a signed event is consumed. Those names are an internal contract, not a promise that every provider uses identical vocabulary. The adapter translates external events into the smaller state machine and stores the original event for audit.

A warning should fire when the oldest `queued` record approaches the queue-delay objective. Another should fire when accepted records stop receiving terminal events, partitioned by sending domain rather than only by account. This gives the on-call time to inspect queue lag, callback verification, DNS authentication, and a recent template or routing change before the customer-facing SLO burns quickly enough to page.

The numerator for the delivery indicator is welcome messages that reach the chosen success state within the objective window. The denominator must exclude deliberate suppressions defined by policy, but it must not quietly discard bounces, expired retries, or records with missing events. Write those exclusions down. Otherwise a dashboard can report perfect delivery while players wait.

That distinction matters.

Opens do not belong in this SLI. Apple Mail Privacy Protection prevents senders from learning a recipient's IP address and privately downloads remote content in the background, so an open event cannot reliably prove that a person read a welcome message. Use a product event, such as completing account activation, as a separate funnel signal; do not relabel it as transport delivery.

## Preserve evidence across the contact-routing boundary

The contact form handler should finish after durable intake and deterministic queue selection. It should not wait for a welcome-message provider, and it should not let email failure move a gameplay or billing question into the wrong support queue. A practical routing record contains a generated case ID, player locale, issue class, queue decision, policy version, and the message correlation ID. Sensitive free text stays out of metric labels and logs.

The sender worker then performs one bounded operation: claim a queued intent, submit it with an idempotency key, record the external message identifier, and schedule reconciliation if the result is ambiguous. A timeout is ambiguous. Retrying without a stable key can create duplicates; marking it failed can lose a message the remote service accepted. The data model has to represent `unknown`, even if that state makes the dashboard less tidy.

This focused Go sketch shows the instrumentation boundary. The interface is deliberately generic, and the worker does not claim delivery after submission.

```go
package welcome

import (
    "context"
    "errors"
    "time"
)

type Intent struct {
    ID       string
    PlayerID string
    To       string
    Domain   string
    Template string
    QueuedAt time.Time
}

type Receipt struct {
    MessageID  string
    AcceptedAt time.Time
}

type DeliveryAPI interface {
    SubmitWelcome(ctx context.Context, intent Intent, idempotencyKey string) (Receipt, error)
}

type Store interface {
    MarkAccepted(ctx context.Context, intentID string, receipt Receipt) error
    MarkAmbiguous(ctx context.Context, intentID string, observedAt time.Time) error
}

type Metrics interface {
    ObserveSubmission(domain, outcome string, latency time.Duration)
}

func Send(ctx context.Context, api DeliveryAPI, store Store, metrics Metrics, in Intent) error {
    started := time.Now()
    receipt, err := api.SubmitWelcome(ctx, in, in.ID)
    if err != nil {
        metrics.ObserveSubmission(in.Domain, "ambiguous", time.Since(started))
        if markErr := store.MarkAmbiguous(ctx, in.ID, time.Now()); markErr != nil {
            return errors.Join(err, markErr)
        }
        return err
    }

    metrics.ObserveSubmission(in.Domain, "accepted", time.Since(started))
    return store.MarkAccepted(ctx, in.ID, receipt)
}
```

The metric dimensions are intentionally sparse. `domain` and `outcome` have bounded cardinality; player IDs, recipient addresses, case IDs, and provider message IDs belong in traces or records with controlled retention. A histogram of submission latency helps diagnose the API boundary, while gauges for oldest queued age and oldest accepted-without-event age expose backlog. Counters for normalized outcomes support the SLI. None of those measurements alone establishes inbox placement.

## Make domain authentication a release gate

Domain authentication is a deployment gate. Verify the exact sending domain, publish and validate its SPF policy, validate the DKIM selector used to sign outgoing mail, and confirm that a real message preserves the expected authenticated identity. DNS presence is not enough: the application, signing configuration, and visible sender must agree with the policy you intend to operate. Run that check before directing production traffic to a new domain or selector, then continue probing it because DNS and signing configuration can change independently of application code.

## Assign the operating burden before procurement

Delivery reliability is the primary decision axis, but reliability includes the operational surface handed to the platform team. A managed API can remove mail-transfer operations while leaving queue semantics, consent evidence, support routing, data retention, and incident response firmly in your system. Self-hosting can increase control, yet it also puts sender reputation work, abuse handling, bounce processing, and on-call ownership onto the same team.

The trade-off has hard boundaries. A managed API is not suitable when policy requires the organization to operate the entire transfer path or when the service cannot export the event evidence required for reconciliation. A self-hosted path is a poor fit for a small platform rotation that cannot staff abuse response, reputation monitoring, key rotation, upgrades, and delivery incident handling. An API-only integration also has a limitation: during a provider control-plane outage, the application cannot fall back to an unrelated SMTP route unless that second path was deliberately designed, authenticated, capacity-tested, and exercised. Adding such a fallback can reduce one dependency while creating duplicate-delivery risk and a second system whose reputation and event semantics must be operated. I prefer one well-tested path plus durable queuing until the recovery objective demands more, because an untested escape hatch is inventory, not resilience.

| Decision surface | Managed delivery API | Self-hosted delivery path | Acceptance evidence |
|---|---|---|---|
| Submission | External API availability and quotas are dependencies | Transfer capacity and upgrades are internal duties | Load test with retries, timeouts, and duplicate attempts |
| Authentication | Service signs mail after domain setup | Team operates keys and signing configuration | Automated SPF and DKIM checks plus a received-message test |
| Event trail | Adapter consumes an external event schema | Team designs and runs the complete event stream | Replay test, signature rejection test, and reconciliation drill |
| Lock-in | Provider identifiers and event meanings can leak inward | Infrastructure and operating knowledge become local commitments | Internal state machine and exportable raw events |
| On-call load | Escalation crosses an organizational boundary | Every transport failure lands on the platform rotation | Named owner, runbook, and measured recovery objective |

I would reject either path if it cannot answer three questions from stored evidence: Was this intent durably queued? Was it accepted once under a stable idempotency key? What terminal event, if any, followed? That is a stricter gate than a feature checklist, and intentionally so. It also makes a later migration less theatrical because application code depends on a narrow delivery interface while reconciliation retains the source event.

Consent needs its own ledger. GDPR Article 7 places conditions on consent and requires that a controller be able to demonstrate it where processing relies on consent. A transactional welcome message and an optional marketing stream therefore should not share an unexplained boolean. Store the policy basis, source, timestamp, and version needed by your governance model, and make suppression decisions before submission. Legal interpretation belongs with qualified counsel; the engineering obligation is to preserve and enforce the decision the organization has approved.

## Charge false positives against on-call capacity

Deploy the instrumentation before changing the page. First shadow the candidate alerts, then compare them against queued age, accepted-without-event age, authenticated test delivery, support-case volume, and downstream activation. Exercise a lost callback, a delayed callback, a submission timeout followed by late acceptance, an invalid event signature, and a queue partition that stops draining. The test passes only when replay and reconciliation converge without producing a second welcome email.

Set warning and paging windows from the published delivery objective and measured event latency, not from an aesthetically pleasing round number. The 12-minute page in the opening is example policy, not a universal threshold. A low-volume game may need synthetic probes because a ratio over a handful of messages is noisy; a launch spike needs burn-rate logic that notices rapid budget consumption without paging on one bounce. Keep absolute counts beside ratios.

False positives have a capacity cost. If every delayed callback wakes someone, responders learn to distrust the signal, and the alert consumes the same constrained on-call hours needed for database, gameplay, and account incidents. If the threshold is too forgiving, support agents become the monitoring system and players discover the gap first. The defensible setting is the one validated against your traffic distribution, event delay, retry horizon, and error-budget policy, with a warning early enough to act and a page reserved for credible SLO impact.

The durable design is plain: authenticate the domain, decouple intake from delivery, retain a correlation trail, reconcile ambiguous submissions, and alert on aging user-visible state. Choose the operating model that can prove those behaviors under failure. Everything else is procurement detail.

## Further reading

- Apple, "Use Mail Privacy Protection on iPhone": https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- GDPR Article 7, "Conditions for consent": https://gdpr-info.eu/art-7-gdpr/
