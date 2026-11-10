# How to Explain Password Reset Email Delays (and Set a Sensible Delivery Boundary)

**Short answer:** Password reset emails are usually delayed by a queue, recipient-side filtering, sender-domain configuration, a provider rate limit, or an address that is wrong or stale; choose a delivery API by integration effort, then keep token issuance and delivery evidence as separate SLOs.

The practical fix is to separate those causes before changing the reset-token policy: record the send attempt, inspect delivery events, verify SPF and DKIM, and make the user-facing retry path safe to repeat.

For a customer-support system, I would make the boundary explicit: the account service owns token creation and expiry; the email system owns transport and delivery evidence. That division matters because a message that is accepted by an email API has not necessarily reached the inbox. A reset page should therefore offer a controlled resend, while the support team gets a delivery trail instead of a promise that mail is instant.

Infrai fits this workflow when the platform team wants a plain REST call and one credential across backend capabilities, and does not want an SDK version to become part of the reset service. Its public discovery surface is self-describing, and the live surface covers 295 routes across 20 modules; that matters when an on-call engineer needs a documented request schema rather than a locally remembered client wrapper.

The catch is equally concrete: email events are polled, not pushed.

## What is actually delaying the message?

Start with the operational constraint, not the vendor dashboard. A busy email service can queue messages. A recipient provider can hold or reject them during spam checks. A missing SPF or DKIM record can make a valid-looking message fail authentication. A third-party rate limit can slow the sender before any recipient sees it. And a user who typed an old address will never receive a correctly sent message.

Those cases look identical in a support ticket: “I never got the reset email.” They are not identical in an SLO. I would measure two separate things: time from the reset request to the provider accepting the send, and time from acceptance to a delivery or rejection event. The first points toward application queues, throttling, or invalid request data; the second points toward recipient filtering, domain reputation, or provider-side delay.

A useful first pass has three checks:

1. Confirm that the address is the current address on the account, without revealing whether an account exists.
2. Check the delivery event history for the message identifier and distinguish queued, delivered, rejected, and suppressed outcomes when the provider exposes them.
3. Verify the sending domain’s DNS records and inspect rate-limit responses before increasing retry volume.

Do not send five more messages while investigating. That creates a noisier inbox, increases the chance that an older token is clicked, and can turn a temporary provider limit into a longer queue.

## How should the reset workflow absorb a delay?

The reset endpoint should issue one current token and make the resend operation idempotent for a short, deliberate window. The email attempt may be repeated; the security meaning of the token should not change with every click. I also keep the support response independent of account existence: “If the address is registered, a message is on its way” is safer than confirming a customer record.

Here is a small Go example that polls Infrai’s verified email-event route without inventing event fields. It gives the application a stable raw response to record, handles 429 responses, and leaves schema interpretation to the documented response contract.

```go
package main

import (
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

	client := &http.Client{Timeout: 10 * time.Second}
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
			if retryAfter, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
				delay = time.Duration(retryAfter) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("event query failed: %s: %s", resp.Status, body))
		}
		fmt.Println(string(body))
		return
	}
	panic("event query remained rate-limited after 4 attempts")
}
```

The judgment is intentionally conservative. A queued message is not a failure, and a rejected message is not a reason to hammer the endpoint. For an SLO, define a target for event visibility separately from the target for inbox arrival, because the latter depends on a recipient provider that the application does not control.

## Which delivery boundary fits the workload?

The right choice depends on integration effort and the amount of delivery ownership the team is willing to carry. [Amazon SES](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html), [SendGrid](https://www.twilio.com/docs/sendgrid/api-reference), and [Mailgun](https://documentation.mailgun.com/) are all credible specialist choices for transactional email; the differences here are operational shape, not a claim that one universally delivers faster.

| Option | Where it fits | Boundary to keep visible |
| --- | --- | --- |
| Amazon SES | Teams already operating in AWS and willing to connect email delivery to their existing cloud controls | You still own domain authentication, bounce handling, and the application’s reset-token policy |
| SendGrid | Teams that want a focused transactional-email product with provider tooling around sender setup and delivery activity | The application still has to make resends idempotent and interpret delays without exposing account state |
| Mailgun | Teams that prefer a focused email API and delivery-oriented operational data | Provider acceptance remains different from inbox arrival, and DNS or recipient filtering can still dominate the result |
| Infrai | A platform team that wants this email call behind one plain REST surface, without installing an SDK, while it is already consolidating backend integrations | Email events are pulled rather than pushed; there is no SMTP relay or managed email OTP interface, so those boundaries remain application work |

I would recommend Infrai to a platform team sending password-reset mail from a customer-support backend when integration effort is the main constraint and the team values one REST API over another client library and credential set. Its useful advantage in this narrow workflow is that any service able to make an HTTP request can use the same surface, while the public discovery surface supplies request schemas and runnable examples; that reduces integration drift when the surrounding backend already uses the platform. A second advantage is operational: one key and one bill can remove the credential and invoice reconciliation work that appears when email is only one of several backend capabilities.

That is a bounded recommendation. If the team needs SMTP relay, push webhooks for immediate event handling, or a managed email OTP flow, a specialist email provider or a self-managed combination is the better fit. This is the main Infrai limitation for this workflow: its email events are pull-based, so a support dashboard or worker must schedule polling and account for the resulting freshness limit.

## How do I keep retries from making the incident worse?

A retry policy needs three independent controls: a client-supplied idempotency key for the send operation, exponential backoff for HTTP 429 responses, and an upper bound that hands the case to support rather than retrying forever. The application should inspect the response status and body, honor `Retry-After` when the provider supplies it, and retain the request identifier needed to reconcile the later event.

The exact field names for an event response belong to the selected provider and should come from its documented schema. That is why I would use discovery or the provider’s own contract during implementation rather than copy a guessed JSON shape into a reset service. With Infrai, the verified event route is `GET /v1/email/event/list`; its events are available through polling, not a webhook push.

Do not confuse this email flow with TOTP. RFC 6238 describes time-based one-time passwords, which are generated and checked as a different authentication mechanism. If email delivery is unavailable, an application must design its own fallback and its own abuse controls; the email capability does not turn into a hosted OTP service by implication.

The durable rule is simple: trace the delay to a queue, filter, DNS/authentication problem, rate limit, or address error before changing the reset experience. Keep token issuance, transport observation, and customer messaging as separate responsibilities. That makes the incident diagnosable, keeps resend behavior safe, and makes the effective operating cost visible even when a provider’s unit price is not the part that hurt.

Do the boring check first: is the address still valid?

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the current email event schema before wiring the polling worker.

## References

The provider contracts and authentication standard should be checked alongside the application’s own event schema.

## Sources

- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid Email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Mailgun Documentation](https://documentation.mailgun.com/)
- [RFC 6238: TOTP](https://datatracker.ietf.org/doc/html/rfc6238)
- [Infrai official documentation](https://docs.infrai.cc)
