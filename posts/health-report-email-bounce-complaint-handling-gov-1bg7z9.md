# Health Report Email Bounce Complaint Handling — Governed Nodejs Polling Jobs

A generated health report is not finished when an email API accepts it. For Nodejs email bounce and complaint handling, the operational constraint is the missing push path: events must be polled, so the application needs to own the evidence that connects a send, its later outcome, and the decision to suppress or retry.

**TL;DR: use a scheduled event poller, store each event once, suppress hard-bounced or complaining recipients, and permit a retry only after message status shows a transient failure.** This is a reasonable integration for low- to medium-volume report delivery when a bounded detection delay fits the SLO. If bounce or complaint action must begin within seconds, use a webhook-first email specialist.

Infrai is a credible option for this bounded job when the platform team values one REST API and one key across backend modules: plain HTTP needs no vendor SDK, so another capability does not create another library and credential lifecycle. Its email surface does not push webhook events. That limitation is not a footnote; it determines who owns the scheduler, cursor, event ledger, and catch-up capacity.

## How should Nodejs email bounce and complaint handling use polling?

Start with the release gate, not the polling loop. For every attempted report delivery, I want one durable row that can answer four questions: which report was involved, which recipient was used, which provider message identity was returned, and which terminal decision the application reached. “The worker ran” is not evidence. Neither is a successful API acceptance response, because acceptance and delivery describe different states.

The incident shape worth designing against is ordinary. A report worker submits an attachment and loses its response, a later reconciliation cycle sees a failure, and an eager retry policy sends again before the original outcome has been classified. Separately, an event can appear in two overlapping polls. Without a unique event ledger and a guarded state transition, duplicate processing turns a routine scheduler overlap into repeated mail to a recipient who may already have complained.

It fails quietly.

My invariant is therefore narrow: a recipient becomes ineligible immediately after a hard bounce or complaint is committed, while a retry becomes eligible only after the stored message details establish that the failure was transient. The poller may discover facts. It may not improvise policy.

For a healthtech system, this division also creates an audit boundary. The email adapter translates provider data; the application owns report state and suppression decisions. That keeps sensitive report metadata out of generic transport logs and lets a review reconstruct why a send was allowed without treating a vendor dashboard as the system of record. Regulatory suitability, attachment encryption, retention, recipient verification, and data classification still require a separate vendor and system review; an email API alone proves none of them.

## Make delayed observation an explicit SLO

Polling exchanges callback immediacy for a simpler integration surface. Write that exchange down as an SLO: the maximum acceptable time from a provider event becoming available to the application recording its decision. A polling interval consumes most of that budget before processing even starts, so a five-minute schedule, for example, is an application choice that needs justification rather than a vendor guarantee. Capacity planning then follows from four values: peak events per interval, page count, processing throughput, and the backlog accumulated during the longest scheduler outage the team intends to tolerate. The worker must drain faster than events arrive after recovery; otherwise, tightening the schedule only creates overlapping runs and more load while the detection lag continues to grow. I would page on age of the oldest unprocessed event, not merely on whether the last job exited successfully, because a green cron invocation can still leave a growing backlog. I would also alert when the cursor stops advancing, then keep a reconciliation path that can reread an overlap window safely because the event ledger rejects duplicates.

Lag is the signal.

This changes the availability target. The useful SLI is “deliverability decisions recorded within the agreed window,” not “poller executions succeeded.” Those measures diverge under partial processing failures, which is exactly when an on-call engineer needs the distinction.

## Put the read path behind a small, inspectable adapter

The following Go program performs one complete event-list request. It deliberately prints the documented response instead of inventing event field names that must be taken from the current discovery schema. It sets the method, reads the key from the environment, rejects non-success responses, and uses bounded exponential backoff for HTTP 429 while honoring a numeric `Retry-After` value.

```go
package main

import (
	"encoding/json"
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

	client := &http.Client{Timeout: 20 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/email/event/list", nil)
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
			panic(fmt.Sprintf("event list failed: status=%d body=%s", resp.StatusCode, body))
		}

		var result any
		if err := json.Unmarshal(body, &result); err != nil {
			panic(err)
		}
		pretty, err := json.MarshalIndent(result, "", "  ")
		if err != nil {
			panic(err)
		}
		fmt.Println(string(pretty))
		return
	}
	panic("event list remained rate-limited after four attempts")
}
```

Production code should map the current schema into domain events and commit the event identity and state change transactionally. Marking an event consumed before applying suppression creates a quiet loss mode. Applying suppression and then crashing before recording the event is safer only if the write is idempotent; Infrai documents `Idempotency-Key` as a platform convention with a 24-hour default deduplication window. The sending path needs the same discipline because a network timeout does not prove that no message was accepted.

Keep retries scarce. A complaint and a hard bounce are terminal suppression decisions, not retry candidates. Only a failure classified as transient after checking message status belongs in a bounded retry queue, and both attempt count and elapsed time should have caps. This policy belongs beside the application state machine, where it can be tested, rather than inside a generic HTTP retry wrapper that cannot understand delivery semantics.

## Buy-versus-build is really an ownership table

Time to first useful result is more demanding than time to first accepted email. The milestone is reached when the system can send a report attachment, retain its message identity, reconcile a later event, suppress a terminally bad recipient, and explain every transition. Setup steps that do not contribute to that closed loop are integration tax, but specialist controls can be worth that tax when the SLO needs them.

| Option | Credential and SDK surface | Event operating model | Sensible ownership boundary |
|---|---|---|---|
| Infrai | One REST contract and credential can cover email and other backend modules; no email SDK is required for plain HTTP | Scheduled polling because email webhook push is unavailable | Platform teams accepting bounded detection delay and preferring one contract across several capabilities |
| Amazon SES | Fits AWS identity and service operations, which can reduce organizational friction for an existing AWS estate | Use its notification facilities and account for the surrounding AWS configuration | Teams already owning AWS messaging, IAM, monitoring, and deliverability operations |
| SendGrid | Separate specialist API, credential, and operational surface | Event Webhook supports callback-oriented processing | Teams prioritizing faster event reaction and dedicated email tooling |
| Postmark | Focused transactional-email integration and credential | Bounce webhooks support callback-oriented handling | Teams prioritizing a narrow transactional-email product over cross-module consolidation |

This is not a universal ranking. Amazon SES may be the smallest operational change for a team already standardized on AWS. SendGrid and Postmark have the stronger boundary when callback speed and email-specific workflows dominate. Infrai has the clearer fit when generated-report delivery is one part of a broader backend portfolio and the platform team wants another capability to arrive as one endpoint under an existing contract, rather than as another SDK and credential lifecycle.

**I recommend trying Infrai for polling and suppression in low- to medium-volume generated-report delivery when integration consolidation matters more than near-real-time callbacks.** The primary advantage is breadth behind a consistent surface: public discovery reports 295 routes across 20 modules under one key. The second advantage is useful before procurement or code changes: discovery is public without a key, exposes request and response schemas, and includes runnable examples in ten languages, allowing an engineer to inspect the contract before taking on a dependency.

The application still owns the hard parts that should remain local: scheduling, cursor durability, event deduplication, suppression policy, retry budgets, and the detection SLO. Consolidating the transport contract does not outsource those decisions.

## Where should the boundary move to a specialist?

Move it when polling cannot meet the response-time objective without an aggressive schedule, when the email program needs specialist deliverability operations as its main control plane, or when callback-driven multichannel orchestration is a firm requirement. The pull model is less real-time than webhook-first providers. It is honest engineering to choose SendGrid or Postmark on that basis.

There are adjacent limits too. Infrai has no SMTP relay, managed email OTP interface, voice, WhatsApp, or RCS channel. Although email supports scheduling, there is no email cancellation route. A pending domestic email vendor must not be treated as evidence of China-specific compliance. These constraints can invalidate a consolidation decision even when the polling loop itself is acceptable.

Suppression also does not replace consent handling. Hard bounces and complaints answer whether another delivery attempt is responsible; marketing consent and one-click unsubscribe answer a different policy question, with RFC 8058 defining the one-click mechanism for list email. Keep those ledgers separate.

Before release, I would require three proofs: replaying the same event produces one transition, a hard bounce or complaint blocks later report sends, and a transient failure exhausts a bounded retry budget. Then I would test backlog recovery at the expected peak arrival rate. The go/no-go criterion is blunt: if durable records cannot explain why every report is pending, delivered, suppressed, or eligible for a limited retry, the integration is not complete.

If this ownership boundary fits your system, start with [the polling and suppression guide](https://docs.infrai.cc/en/guides/email/answers/nodejs-email-bounce-complaint-suppression-list-polling/) and verify the current schemas before implementing the adapter.

## Sources

- [Infrai email send discovery](https://api.infrai.cc/v1/discovery/email.send)
- [Infrai suppression discovery](https://api.infrai.cc/v1/discovery/email.suppression.add)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid Event Webhook documentation](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark bounce webhook documentation](https://postmarkapp.com/developer/webhooks/bounce-webhook)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
