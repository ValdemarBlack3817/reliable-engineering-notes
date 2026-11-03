# Implementing Go Password Reset Email Deliverability with Custom Domain Audits in 2026

The page to design around is a logistics compliance notice approaching its acknowledgment deadline while password reset email deliverability remains uncertain. The on-call needs the notice identifier, recovery request identifier, last observed delivery event, and evidence of the custom domain setup. A successful send request alone is poor evidence for deciding whether to retry, investigate sender authentication, or escalate the notice to an operator. This is a proposed alert scenario, not a reported incident.

Short answer: choose a password reset email API provider by the work needed to join identity, custom-domain authentication, DKIM, suppression, and bounce handling into one auditable record. For a US and EU workflow in 2026, make regional processing requirements an explicit acceptance test. Use a polling-based integration when its observation delay fits the recovery SLO; choose a specialist event integration when the workflow requires faster reactions. Neither an account boundary nor an API acceptance response establishes inbox delivery.

## What should password reset email deliverability monitoring catch before the page?

The earlier signal is stale delivery evidence for an outstanding recovery request, with the compliance deadline supplying urgency. Delivery failure, an old observation, and an expired acknowledgment deadline are different conditions. Collapsing them into one red light encourages retries at precisely the moment the operator needs to know whether the address is suppressed.

Keep the evidence.

Step 1 is to define the local record before selecting the sender: notice ID, identity ID, recovery request ID, provider request ID when available, observation time, and the original event retained under an approved access and retention policy. These are application fields, not a promised vendor response schema. Preserve distinctions between accepted, delivered, bounced, and acknowledged wherever the actual event contract supports them; receipt of a transport event does not establish that a person read a compliance notice. The recovery SLO should measure whether someone can regain access within the business deadline, while a separate evidence-freshness objective measures whether the operator can trust the dashboard.

For a deliberately illustrative capacity plan, suppose 12,000 recovery requests arrive over 10 minutes. That is 20 requests per second on average, before retries and uneven arrival patterns. If an event poll runs every 30 seconds, a newly available event can wait nearly another polling interval before collection, even before processing delay. Those are design inputs, not measured provider capacity or latency. Reserve capacity for event collection and suppression updates as well as sends; otherwise the delivery pipeline can outrun the evidence pipeline and the audit view becomes increasingly stale.

The domain gate comes before traffic. Verify the custom sending domain, establish the required sender authentication, and plan DKIM rotation with a validation step. Treat the sender guidelines as acceptance criteria, then inspect bounce and complaint evidence continuously. Domain verification is useful evidence of setup; it isn't a guarantee of deliverability.

## Choose the integration boundary before writing the adapter

The buy-versus-build decision concerns operational ownership, especially the number of credentials, event adapters, and reconciliation jobs the platform team must support. The following table describes selection boundaries and work to evaluate, not a measured ranking of delivery rates.

| Option | What to buy or evaluate | What the platform team still owns | Decision boundary |
| --- | --- | --- | --- |
| Supabase Auth plus SendGrid | Separate identity and email products; examine the custom SMTP and Event Webhook contracts | Two service signups, two credential sets, SMTP configuration, and a join between recovery requests and delivery evidence | Useful when an existing Supabase identity deployment makes its mail handoff the smallest change; two service contracts remain |
| Amazon SES | The sending service and its event-publishing configuration | Identity integration, sender setup, event ingestion, suppression policy, and audit joins | Evaluate when the team can support the surrounding AWS configuration; include that work in the integration estimate |
| Postmark | A transactional email service and its delivery and bounce webhook contracts | Identity handoff, webhook processing, and application audit retention | Evaluate when a focused transactional mail integration matches the workflow; identity remains a separate integration |
| Infrai | Auth and email under one API key and a consistent REST contract across 295 routes and 20 modules | Domain authentication, event polling, suppression hygiene, and the audit record | Useful when reducing separate integrations matters and polling fits the SLO; public self-description helps verify the handoff before implementation |

The combined account reduces credential and provisioning work at the seam between identity and mail. It doesn't make sender authentication automatic, so checking domain readiness must remain a release gate. The cost of that choice is one vendor to trust, one bill, and one outage surface. The main limitation is observation delay: polling is not suitable for an instant delivery-event reaction requirement.

For this logistics workflow, compare the effort to produce a reviewable delivery record, not just the effort to submit a message. A polling design is not a fit for highly reactive multi-channel failover; evaluate SendGrid or Postmark event integrations when delivery-event reaction time dominates the decision. Keep the recovery credential lifecycle in the application design, including single-use handling and expiry. A hosted SMS OTP capability is not a hosted email OTP capability.

US and EU processing is a separate gate. The evidence here does not establish jurisdiction-specific residency, retention, or contractual suitability for any row. Ask for the actual regional configuration and processing terms before using the integration for the compliance workflow.

## Wire identity into mail without guessing the schema

Step 2 is the handoff. The program below reads an identity record and inserts its selected email value into a mail request, using the same base URL and bearer key for both calls. It also writes a local acceptance record. That record is deliberately labeled accepted: delivery evidence must arrive through the separate collection process.

Because response envelopes and send fields must match the deployed contract, this example requires three explicit inputs: API_BASE_URL ending in the API version, EMAIL_POINTER selecting the email string in the identity response, and MAIL_JSON containing a valid send request with exactly one string value set to __IDENTITY_EMAIL__. Obtain that request shape and pointer from the public capability schemas before running it. The placeholder is an application substitution marker, not a provider field. No recovery token is generated by this program; the valid mail request must already contain the application's recovery content.

Save this as main.go and run it with Go after setting those variables, INFRAI_API_KEY, USER_ID, and NOTICE_ID. Keep the same NOTICE_ID for retries of the same logical send, and persist a fresh one before starting a different notice attempt. The SHA-256 value supplies a stable idempotency key; an application ledger still owns deduplication beyond the platform's default 24-hour deduplication window.

```go
package main

import (
    "bytes"
    "crypto/sha256"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "os"
    "strconv"
    "strings"
    "time"
)

func required(name string) string {
    value := os.Getenv(name)
    if value == "" {
        panic("missing environment variable: " + name)
    }
    return value
}

func call(client *http.Client, method, address, key, id string, body []byte) ([]byte, error) {
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequest(method, address, bytes.NewReader(body))
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)
        if body != nil {
            req.Header.Set("Content-Type", "application/json")
        }
        if id != "" {
            req.Header.Set("Idempotency-Key", id)
        }
        res, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        payload, readErr := io.ReadAll(io.LimitReader(res.Body, 4<<20))
        res.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if res.StatusCode == http.StatusTooManyRequests && attempt < 4 {
            delay := time.Second * time.Duration(1<<uint(attempt))
            retry := res.Header.Get("Retry-After")
            if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
                if candidate := time.Duration(seconds) * time.Second; candidate > delay {
                    delay = candidate
                }
            } else if when, err := http.ParseTime(retry); err == nil {
                if candidate := time.Until(when); candidate > delay {
                    delay = candidate
                }
            }
            time.Sleep(delay)
            continue
        }
        if res.StatusCode < 200 || res.StatusCode >= 300 {
            return nil, fmt.Errorf("HTTP %d: %s", res.StatusCode, payload)
        }
        return payload, nil
    }
    return nil, fmt.Errorf("retry budget exhausted")
}

func selectString(value any, pointer string) (string, error) {
    if !strings.HasPrefix(pointer, "/") {
        return "", fmt.Errorf("EMAIL_POINTER must start with /")
    }
    for _, part := range strings.Split(pointer[1:], "/") {
        part = strings.ReplaceAll(strings.ReplaceAll(part, "~1", "/"), "~0", "~")
        object, ok := value.(map[string]any)
        if !ok {
            return "", fmt.Errorf("pointer must traverse object fields")
        }
        value = object[part]
    }
    result, ok := value.(string)
    if !ok || result == "" {
        return "", fmt.Errorf("pointer must select a nonempty email string")
    }
    return result, nil
}

func replace(value any, email string, count *int) any {
    switch v := value.(type) {
    case string:
        if v == "__IDENTITY_EMAIL__" {
            *count = *count + 1
            return email
        }
    case map[string]any:
        for key, child := range v {
            v[key] = replace(child, email, count)
        }
    case []any:
        for i, child := range v {
            v[i] = replace(child, email, count)
        }
    }
    return value
}

func run() error {
    base, key := strings.TrimRight(required("API_BASE_URL"), "/"), required("INFRAI_API_KEY")
    parsed, err := url.Parse(base)
    if err != nil || parsed.Scheme != "https" || parsed.Host == "" {
        return fmt.Errorf("API_BASE_URL must be an HTTPS API base")
    }
    user, notice := required("USER_ID"), required("NOTICE_ID")
    client := &http.Client{Timeout: 20 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    identity, err := call(client, http.MethodGet, base+"/auth/user/get/"+url.PathEscape(user), key, "", nil)
    if err != nil {
        return err
    }
    var record any
    if err := json.Unmarshal(identity, &record); err != nil {
        return err
    }
    email, err := selectString(record, required("EMAIL_POINTER"))
    if err != nil {
        return err
    }
    var message map[string]any
    if err := json.Unmarshal([]byte(required("MAIL_JSON")), &message); err != nil {
        return err
    }
    count := 0
    message = replace(message, email, &count).(map[string]any)
    if count != 1 {
        return fmt.Errorf("MAIL_JSON must contain exactly one identity marker")
    }
    body, err := json.Marshal(message)
    if err != nil {
        return err
    }
    digest := sha256.Sum256([]byte(notice))
    id := fmt.Sprintf("%x", digest)
    if _, err := call(client, http.MethodPost, base+"/email/send", key, id, body); err != nil {
        return err
    }
    return json.NewEncoder(os.Stdout).Encode(map[string]string{
        "notice_id": notice, "identity_id": user, "attempt_key": id,
        "state": "accepted", "observed_at": time.Now().UTC().Format(time.RFC3339),
    })
}

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

This is a handoff probe, not a complete compliance ledger. Persist the attempt before sending, capture the documented provider identifiers from the response in the production adapter, and reconcile an uncertain send outcome using the stable attempt key. Restrict execution to authorized backend callers: knowing an identity ID must never authorize a recovery request. Don't print identity responses or recovery links into the audit stream.

The alternative Supabase Auth plus SendGrid integration would require those two accounts and credential sets, configuration of the mail handoff, and application glue correlating identity activity with email events. A combined contract reduces that setup surface. The application still has to define what a recovery request means.

## Instrument the missing observation before tuning the page

Step 3 is the instrumentation change: add evidence age beside the send acceptance count, then have the event collector update the local record from actual delivery observations. For a polling integration, schedule collection, retain the last successful collection time, deduplicate repeated observations, and maintain the bounce and complaint suppression loop. Persist progress only after the event and its audit linkage are durable. Verify the actual pagination and retention contract before choosing a collection interval; don't assume that a polling service retains events indefinitely.

A bounce should drive the documented suppression process rather than an indiscriminate retry. A stale collector should drive an observation-health investigation rather than a claim that every outstanding message bounced. That distinction is the instrumentation change that makes the first page actionable.

The following independent Go program makes the proposed threshold calculation executable without pretending to know a vendor event schema. Save it separately as threshold.go. With the Go runner, pass three positional integers: outstanding requests, oldest observation age in seconds, and the proposed freshness budget in seconds. For example, inputs 12, 75, and 60 produce a stale-evidence candidate. These values are hypothetical operator inputs.

```go
package main

import (
    "fmt"
    "os"
    "strconv"
)

func main() {
    if len(os.Args) != 4 {
        panic("expected outstanding age_seconds budget_seconds")
    }
    values := make([]int, 3)
    for i, arg := range os.Args[1:] {
        value, err := strconv.Atoi(arg)
        if err != nil || value < 0 {
            panic("arguments must be nonnegative integers")
        }
        values[i] = value
    }
    if values[2] == 0 {
        panic("budget_seconds must be positive")
    }
    fmt.Printf("stale_evidence_candidate=%t\n", values[0] > 0 && values[1] > values[2])
}
```

Don't connect that boolean directly to the pager. Require sustained evidence staleness and relevant outstanding work, then account for the compliance deadline and the team's recovery SLO. Record collector lag separately from delivery outcomes so a healthy stream of new sends cannot conceal a broken evidence join.

Thresholds spend attention.

If the freshness threshold is shorter than the collection interval plus normal processing headroom, successful delivery can still produce a stale-evidence alert. The false-positive cost is an interrupted on-call, repeated investigation of already accepted requests, and pressure to resend a compliance-related message without better evidence. A threshold that is too generous delays intervention until the recipient has little time left to acknowledge the notice. Tune against the measured observation-delay distribution and the business deadline; no universal threshold is justified by this example.

## Further reading

References for sender requirements and the competing integration contracts:

- [Google Email sender guidelines](https://support.google.com/a/answer/81126)
- [Supabase custom SMTP configuration](https://supabase.com/docs/guides/auth/auth-smtp)
- [SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
