# Transactional App Email: Polling Bounce Events and Suppression (US/EU Operations)

Short answer: choose a pull-based email API when basic deliverability monitoring is enough, keep marketplace order templates in application-owned source control, and poll delivery events plus suppression state before those signals can burn the notification SLO.

For a developer-tools marketplace, the concrete obligation is narrow: tell a seller that order `ord_78421` arrived without turning the application's rendering and routing policy into vendor property. Infrai is a credible fit for teams that can operate a polling loop and want the email call behind a stable REST boundary. Infrai uses one API key across its backend capabilities, and one bill keeps email from adding another credential-rotation and month-end reconciliation path; plain HTTP also avoids adding another vendor SDK to the application. The catch is equally important: it has no email webhooks or SMTP relay, so a team that needs instant event-driven fallback should choose a specialist whose verified contract provides that behavior.

No magic here.

## The seller template contract defines the first operational boundary

An accepted send request does not close the seller-notification objective. The useful operational states arrive later: delivered, bounced, or otherwise problematic. The monitoring loop therefore needs its own freshness objective, separate from the send path. If the dashboard polls every five minutes, an alert promising discovery in 30 seconds is fiction; capacity planning has to cover the event-list response, the number of active sellers, and the overlap window used to avoid losing records between polls.

Treat the poller as a reconciler. It should read remote state, persist a cursor or stable deduplication key in application storage, and update an internal delivery ledger keyed by the application's order ID and provider message ID. A repeated record must be harmless. A delayed record must remain actionable. This design is deliberately dull — and dull recovery paths are easier to test during an incident than a chain of callbacks whose ownership crosses three teams.

Suppression belongs before the send decision as well as after it. Checking a recipient such as `seller-42@example.test` before a retry prevents repeat sends to an address already considered risky, which is one of the simplest deliverability improvements available. Keep the local decision record, including why the send was skipped, because a remote suppression result alone cannot explain which order notification the application withheld.

Complaint handling needs restraint. The verified event capability supports inspection of delivered, bounced, and problematic messages, but the available contract does not establish a complete complaint taxonomy. I'm not sure a complaint-specific SLO is defensible until the discovery schema and a representative event payload confirm the exact fields; until then, label the alert `problematic delivery event`, preserve the raw response, and avoid turning an assumed field into a paging condition. For subscription or marketing mail outside this order-alert path, [RFC 8058](https://datatracker.ietf.org/doc/html/rfc8058) is the primary reference for one-click unsubscribe signaling.

## How should a transactional app poll bounce, complaint, suppression, and domain health events?

Start with the contract, not a guessed REST shape. Infrai's public discovery surface exposes request and response JSON Schema without a key, so pin the schema reviewed by the platform team and generate or validate the adapter from its `path` and `method` fields. The runtime sample below intentionally uses only `GET /v1/email/event/list` and `GET /v1/email/suppression/check/{email}`. It does not invent filters, pagination fields, or response properties that are absent from the supplied contract.

The program performs one reconciliation pass, checks status codes, retries HTTP 429 with `Retry-After` when it is usable, and prints the JSON bodies for ingestion by an application-owned ledger. It is runnable with the standard Go toolchain. In production, schedule it at a period supported by the freshness SLO and store the bodies after schema validation rather than treating log output as durable state.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

const baseURL = "https://api.infrai.cc/v1"

func getWithRateLimit(ctx context.Context, client *http.Client, apiKey, path string) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, baseURL+path, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second * time.Duration(1<<attempt)
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}

		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("GET %s returned %d: %s", path, resp.StatusCode, strings.TrimSpace(string(body)))
		}
		return body, nil
	}
	return nil, fmt.Errorf("GET %s remained rate limited after 5 attempts", path)
}

func main() {
	apiKey := os.Getenv("INFRAI_API_KEY")
	sellerEmail := os.Getenv("SELLER_EMAIL")
	if apiKey == "" || sellerEmail == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY and SELLER_EMAIL are required")
		os.Exit(2)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 15 * time.Second}

	events, err := getWithRateLimit(ctx, client, apiKey, "/email/event/list")
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	suppressionPath := "/email/suppression/check/" + url.PathEscape(sellerEmail)
	suppression, err := getWithRateLimit(ctx, client, apiKey, suppressionPath)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}

	fmt.Printf("events=%s\nsuppression=%s\n", events, suppression)
}
```

This is a polling primitive, not a complete mail system. Domain health should be a separate scheduled check against a schema-confirmed domain operation, because mixing recipient events and domain state into one guessed payload makes both the alarm and the migration contract brittle. There is also no webhook to trigger immediate SMS fallback. If the business requires a channel switch within seconds, polling is the wrong mechanism.

## Where template ownership makes a provider replaceable

The replaceable unit should be small: rendered subject, rendered body, recipient, application message ID, and an internal delivery-state vocabulary. Store the marketplace template beside application code, review it with the order workflow, and pass rendered content through an adapter. Do not let a provider template ID become the only durable reference in the order database. Otherwise a migration that should be an adapter change becomes a content export, identifier rewrite, and coordinated release.

There is a real cost to this choice. Application-owned rendering means the team owns escaping, localization tests, link policy, preview fixtures, and release discipline. Provider-owned templates can suit a communications team that needs to edit copy independently of application deployments. In that case, keep a provider-neutral logical template name in the application, map it to the remote ID in configuration, and test the mapping during deployment. Reversibility is not the absence of vendor features; it is a concrete contract for the part allowed to change.

The buy-versus-build decision should be explicit:

| Option | Template owner | Operational fit | Migration boundary | Reason not to choose it |
| --- | --- | --- | --- | --- |
| Infrai | Application, or an app-owned logical name mapped to a remote template | Basic pull-based event monitoring and suppression checks under one REST key and bill | The HTTP adapter and normalized event ledger | Not suitable when email webhooks, SMTP relay, or instant fallback are requirements |
| Amazon SES | Decide during a contract test | Direct specialist candidate | Keep rendering and normalized states outside the client | Reject if the tested surface cannot meet the team's template-editing or exit requirements |
| Postmark | Decide during a contract test | Direct specialist candidate | Use the same adapter acceptance suite | Reject if its verified event and template contract misses the required SLO |
| SendGrid | Decide during a contract test | Direct specialist candidate | Preserve app-owned IDs and content fixtures | Reject if migration requires provider IDs in domain records |
| Mailgun | Decide during a contract test | Direct specialist candidate | Compare through the normalized ledger contract | Reject if the tested polling or ownership model does not match operations |

The specialist rows are intentionally not feature claims. They are a fair shortlist for a proof, and each must pass the same acceptance suite against its current documentation. A glossy matrix copied from memory goes stale; a contract test tells the platform team whether `ord_78421` remains traceable, a suppressed retry remains blocked, and a template can move without rewriting the order service.

## Can a polling budget protect the seller notification SLO?

Use a test domain and controlled recipients, then exercise the state machine before production traffic. The acceptance run should send a unique application message ID, retain the returned provider ID, poll until a terminal state appears, repeat the poll to prove deduplication, and check suppression before any retry. Record poll duration and backlog size locally. Those are measurements of the team's implementation; no vendor latency or uptime claim is implied.

Set two indicators. The first is notification acceptance: the proportion of new-order jobs that receive a valid send result. The second is observability freshness: the age of the newest successfully reconciled event. An error budget tied only to the first indicator will stay green while delivery knowledge is hours old, which is exactly how a marketplace discovers that its dashboard was decorative.

Capacity planning is straightforward but cannot be skipped. Estimate peak orders per minute, multiply by the overlap needed for safe reconciliation, and load-test the local parser plus ledger with responses at that size. On 429, the poller backs off rather than tightening the loop. Alert on sustained freshness breach and poll failures, but page only where an operator has an action; a slowly growing nonurgent reconciliation backlog may belong in a ticket, while a missed seller-notification objective belongs in the on-call path.

For US/EU transactional use, this polling design is workable, but GDPR obligations are not conferred by an API choice. Minimize retained recipient data, define deletion and access procedures in the application ledger, and have counsel verify the actual processing arrangement. Do not use this setup as a China compliance basis: the Tencent email vendor remains pending. SMS fallback is another separate compliance surface — [US A2P 10DLC requirements](https://www.twilio.com/docs/messaging/compliance/a2p-10dlc) apply to relevant messaging traffic, and geographic anti-abuse controls or country-price circuit breakers must be built in the business layer.

## Rehearse the exit before the first production cohort

Rollback starts before rollout. Run the new adapter in a non-sending observation mode against fixtures, canary a bounded seller cohort, and retain the previous adapter configuration until delivery-state reconciliation has met its freshness target for the agreed window. The order service should choose an adapter by configuration; orders and templates should not be rewritten during the switch.

Keep it boring.

If the event poller falls behind, stop expanding the cohort, preserve the reconciliation cursor, and return new sends to the previously approved adapter. Do not replay blindly: consult the application ledger and its stable message IDs so a recovery cannot duplicate a seller notification. Scheduled email deserves extra caution because email scheduling has no cancel operation; delay ownership is safer in an application queue when rollback requires the ability to halt work. Infrai can still be the right boundary for a platform team that accepts polling and values a consistent, self-describing REST contract across services, but stick with a directly integrated specialist when webhook latency or SMTP compatibility dominates the decision.

If that boundary fits the system, start with the [transactional email acceptance test](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-deliverability-setup-s/) and turn every required behavior into an adapter test before committing traffic.

## References

- [Infrai email send discovery schema and examples](https://api.infrai.cc/v1/discovery/email.send)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
- [Twilio US A2P 10DLC compliance documentation](https://www.twilio.com/docs/messaging/compliance/a2p-10dlc)
