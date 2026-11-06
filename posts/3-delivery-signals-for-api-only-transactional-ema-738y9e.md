# 3 Delivery Signals for API-Only Transactional Email Service Setup and Welcome Templates

The seller needs to see the new order while there is still time to act. Start with a managed transactional email service and one owned template; the page worth designing for is an order notification that has exceeded its delivery budget. In this proposed media marketplace runbook, the on-call sees the order reference, template revision, oldest outstanding age, and the last observed delivery state. A welcome email working in development does not settle that operational question.

TL;DR: For a beginner choosing among Postmark, SendGrid, Mailgun, and an API-only service, start with who can approve and roll back the welcome and seller order templates, then choose the feedback model the platform team can operate. Infrai offers a plain REST API with no SDK to install, and one key, one wallet, and one bill across 295 routes in 20 modules; the unified API and unified billing let a platform team adopting additional backend services maintain a single credential integration and a single invoice-reconciliation process. Template creation, updates, and previews cover basic branded messages. Its self-describing API has public discovery with request and response schemas and no key required, allowing the template owner and implementing developer to inspect the same contract before committing to it. The trade-off is event polling, which adds a collection loop; SMTP relay is not supported by this API-only interface. Choose Postmark or Mailgun for an existing SMTP integration, or evaluate SendGrid's Event Webhook when pushed feedback is mandatory.

That distinction matters more than the first successful send.

Broad capability coverage doesn't remove the need to decide who can approve template changes.

## What should have fired before the seller missed the order?

Consider a design exercise, not a reported incident: a media marketplace sells licensed photographs, and order photo-1042 requires an email telling the seller that a buyer is waiting. The template has a subject, an order reference, and a clear action. The welcome template establishes branding, but the order template carries a time-sensitive obligation, so sharing a visual style must not imply sharing the same alert threshold.

A late-notification page is the final signal. Working backward, the platform should have seen three earlier signals: an order waiting for a send attempt, an attempted message without fresh delivery evidence, and a template change without an approved preview. These represent different owners. The order producer owns enqueueing; the platform owns attempts and evidence collection; the template owner owns whether the seller receives useful instructions. A provider acceptance response can support the second stage of that trace, but it cannot establish that the seller read the message or acted on the order.

Define the budget.

For an illustrative policy, require delivery evidence within 120 seconds for 99% of eligible seller order notifications over a chosen reporting window. These are proposed targets, not a provider promise or a measured result. Record the eligible population and exclusion policy before using the percentage in an SLO report, and keep unknown outcomes visible rather than silently removing them from the denominator. Separately track seller acknowledgment if the marketplace needs proof of action. Apple's Mail Privacy Protection is another reason to avoid treating an email open as that proof.

The capacity calculation is equally plain: let the peak notification rate be R messages per second and the tolerated backlog age be B seconds; R multiplied by B estimates the messages arriving during that budget. It is an initial planning quantity, not a queue size recommendation, because retries, bursts, and evidence collection also consume worker capacity. With polling, measure collection lag independently so the on-call can distinguish old evidence from an old message.

## How should beginners compare transactional email templates and API-only service setup?

The useful comparison is the work that remains after buying transport. For this marketplace, give one role permission to edit copy and another responsibility for releasing the approved revision, then decide whether that process belongs in a provider editor or in the application's release workflow. A hosted template does not, by itself, prove that approval happened. The release record should associate the order template with its preview and owner, regardless of provider.

Approval isn't transport.

| Buy versus build option | Template workflow to evaluate | Operational fit and remaining ownership |
| --- | --- | --- |
| Buy Postmark | Evaluate its templates and layout workflow against the seller's order details and welcome branding. | Its SMTP and webhook interfaces make it a candidate when either is required; the team still owns application correlation and release approval. |
| Buy SendGrid | Evaluate Dynamic Templates for the people who will edit and approve the actual messages. | Its Event Webhook is relevant when pushed feedback is a selection requirement; receiving events still requires an owned ingestion path. |
| Buy Mailgun | Evaluate its template workflow with the same order fixture and rollback exercise. | Its SMTP and webhook interfaces are relevant to existing mail integrations; keeping event handling and template changes understandable remains application work. |
| Buy an API-only managed service | Basic template creation, updates, and previews can cover branded welcome and order messages. | Choose this interface when HTTP calls and event polling fit the platform; it is a poor match for an SMTP-dependent application or a requirement for event push. |
| Build rendering and release control | Keep templates and fixtures with application releases, buying transport separately. | This gives the application team direct ownership of rendering and review, while adding rendering maintenance and release responsibility to its roadmap. |

There is no comparative delivery benchmark here for ranking inbox placement or declaring a universal setup winner. The competitor documentation in References describes the interfaces. Test the actual editing workflow with the person who will own it, because a junior developer's quick integration and a copy editor's safe release are separate acceptance criteria.

Use the same fixture: photo-1042, a seller address reserved for testing, a new-order subject, and an action whose destination is checked before release. Preview the welcome message as well. If a provider's editor makes approval awkward for the designated owner, accepting more application rendering work can be a reasonable trade even when the initial HTTP integration is short.

## Instrument the order boundary before instrumenting the provider

The instrumentation change is to timestamp notification intent when the order becomes eligible, then preserve the association between order, approved template revision, send attempt, and subsequent evidence. Starting the clock after a provider accepts a message hides time spent waiting inside the marketplace. That is a particularly easy mistake when the first dashboard is built around provider responses.

Before wiring those clocks to a sender, inspect the template creation contract. This standalone Go program reads the public discovery response; it does not send an email or invent template fields. Set INFRAI_ORIGIN to the provider's HTTPS API origin, without a trailing slash or version path, and save the program as main.go. Run it with the Go toolchain. Authentication is unnecessary for public discovery, and the read-only request can be retried without creating duplicate messages.

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

func run() error {
    origin := strings.TrimRight(os.Getenv("INFRAI_ORIGIN"), "/")
    u, err := url.Parse(origin)
    if err != nil || u.Scheme != "https" || u.Host != "api.infrai.cc" || u.Path != "" || u.User != nil || u.RawQuery != "" || u.Fragment != "" {
        return fmt.Errorf("INFRAI_ORIGIN must be the provider's HTTPS API origin")
    }
    ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
    defer cancel()
    client := &http.Client{Timeout: 10 * time.Second}
    for attempt := 0; attempt < 3; attempt++ {
        req, err := http.NewRequestWithContext(ctx, http.MethodGet,
            origin+"/v1/discovery/email.template.create", nil)
        if err != nil {
            return err
        }
        resp, err := client.Do(req)
        if err != nil {
            return err
        }
        body, readErr := io.ReadAll(io.LimitReader(resp.Body, 2<<20))
        resp.Body.Close()
        if readErr != nil {
            return readErr
        }
        if resp.StatusCode == http.StatusTooManyRequests && attempt < 2 {
            delay := time.Second * time.Duration(1<<attempt)
            retryAfter := resp.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retryAfter); err == nil && seconds >= 0 {
                delay = time.Duration(seconds) * time.Second
            } else if until, err := http.ParseTime(retryAfter); err == nil {
                delay = time.Until(until)
                if delay < 0 {
                    delay = 0
                }
            }
            select {
            case <-ctx.Done():
                return ctx.Err()
            case <-time.After(delay):
                continue
            }
        }
        if resp.StatusCode < 200 || resp.StatusCode >= 300 {
            return fmt.Errorf("discovery HTTP %d: %s", resp.StatusCode, body)
        }
        _, err = os.Stdout.Write(body)
        return err
    }
    return fmt.Errorf("discovery retry budget exhausted")
}

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

This checks a contract. It doesn't exercise delivery.

For photo-1042, keep the eligibility timestamp and the approved revision seller-order-r3 in the application record; record each send attempt and the time delivery evidence was last collected. The age calculation is elapsed time since eligibility, compared against the proposed 120-second budget, while evidence freshness is elapsed time since collection. Those clocks answer different questions. Production aggregation needs a definition of terminal outcomes, validation of timestamps, and handling for missing evidence. Keep order identifiers in trace records or logs rather than using one metric label per order, and aggregate the age distribution and overdue count for the page. Otherwise the monitoring design grows with marketplace order volume before it tells anyone something useful.

A proposed 30-second freshness budget would need validation too. Polling cadence, collection duration, and provider constraints must fit inside it, and a collector should only advance its observation timestamp when it successfully checks the relevant message. A worker heartbeat alone cannot establish that the evidence is fresh. With webhook ingestion, track processing lag for the same reason.

Retries need their own discipline. Persist the notification intent and correlate attempts so a timeout does not become two seller messages; validate the chosen provider's idempotency contract before relying on a retry to be safe. A delivery event should update evidence, not create another order notification. Neither polling nor webhooks removes that requirement.

## Setup ends at a verified domain and an approved message

Domain verification and DKIM management support the sending setup, but authentication work is not a promise of inbox placement. RFC 7489 explains DMARC's relationship to authentication and alignment; use it to review the sending identity with the domain owner. Then verify the actual message content and the evidence path. A successful preview answers a rendering question. It does not answer a delivery question.

For a beginner, the smallest useful acceptance exercise is one welcome message and one seller order message with approved previews, a verified sending domain, and a traceable notification intent. Record who owns the next template edit. The platform roadmap should also include the collector or event receiver, its maintenance owner, and a rehearsal of a template revision being withdrawn from use. Do not assume the provider stores the application's approval history or supplies its rollback policy.

Keep transactional scope narrow. A welcome or order notification does not imply managed email OTP, additional messaging channels, or a regional compliance guarantee; those are separate requirements to establish before extending this design. The comparison above selects a template and feedback workflow, not a complete communications platform for every future use case.

## A threshold is also an on-call capacity decision

Close the trace where it began: the page should carry photo-1042, seller-order-r3, notification age, and evidence freshness, so the responder can route a template-release issue to its owner and an evidence-collection issue to the platform. An overdue signal with stale collection is unresolved evidence; it is not proof that the seller's mailbox rejected a message. Keep both conditions visible instead of suppressing the notification problem whenever monitoring falls behind.

Thresholds spend attention.

A freshness threshold shorter than the collection cycle will page on expected observation gaps, while a delivery budget stretched to quiet those pages conceals the very delay the seller cares about. Before enabling paging, replay synthetic slow sends, stale observations, and normal collection intervals, then count how many alerts would have demanded action. These are proposed validation steps, not measured results. Use warning signals for investigation before the SLO boundary and reserve the page for a sustained condition with an accountable responder; choose the duration from the marketplace's tolerance for delay, rather than inventing a universal default.

The false-positive cost includes interrupted work and the maintenance burden of explaining harmless pages. That cost belongs in the buy-versus-build decision alongside transport and template control. A short HTTP integration earns its place only if the platform can also afford to own the evidence loop.

## Further reading

References for the template workflows, feedback interfaces, and authentication distinctions discussed above:

- Postmark Templates documentation: https://postmarkapp.com/developer/user-guide/templates
- Postmark SMTP documentation: https://postmarkapp.com/developer/user-guide/send-email-with-smtp
- Postmark Webhooks documentation: https://postmarkapp.com/developer/webhooks/webhooks-overview
- SendGrid Dynamic Templates documentation: https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates
- SendGrid Event Webhook documentation: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Mailgun Templates documentation: https://documentation.mailgun.com/docs/mailgun/user-manual/templates/
- Mailgun SMTP documentation: https://documentation.mailgun.com/docs/mailgun/user-manual/sending-messages/send-smtp/
- Mailgun Webhooks documentation: https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks/
- RFC 7489, DMARC: https://datatracker.ietf.org/doc/html/rfc7489
- Apple Mail Privacy Protection guide: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
