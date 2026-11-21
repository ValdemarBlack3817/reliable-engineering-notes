# How to Capture Backend Exceptions with a Simple Error Tracking API (Checkout Noise)

Choose a simple error tracking API only after its captured exceptions can distinguish broken course purchases from expected checkout rejections. A quieter dashboard is a weak acceptance criterion if the filtering also removes the one exception that explains why a student paid but never received access.

Short answer: For a small SaaS Node.js backend, basic exception capture, grouping, stack trace investigation, and search can be enough; decide by the quality of the evidence retained after checkout noise is classified. Buy broader investigation capabilities when browser debugging or distributed tracing is part of the requirement, and assign notification ownership explicitly rather than treating a searchable event as a page delivered.

Keep uncertainty visible.

Basic capture deserves consideration when the platform team needs searchable evidence without adopting an entire observability suite. Keep that scope explicit during selection.

## 1. How should a small SaaS filter exceptions before error tracking API capture?

Start with an operational distinction: an expected payment rejection can be a normal checkout outcome, while an unexpected exception during enrollment can threaten the fulfillment SLO. Neither the HTTP response class nor the text of an exception establishes that distinction reliably. Define the SLO using eligible paid purchases and the promised access deadline; error groups help explain a bad outcome, but they cannot provide the purchase denominator or establish that access arrived on time.

Use an explicit application classification for expected outcomes. Leave an unfamiliar exception in the investigation stream until its meaning is established. A rule that suppresses any message containing “declined” is particularly suspect: the same word can appear in a wrapped failure from another checkout stage, and a string match has no knowledge of the purchase state.

Names are evidence.

The proposed policy below has two independent decisions: retain unexpected exception evidence, then select notification candidates from unexpected failures with a known access impact. An unknown impact remains visible for review. It does not become harmless merely because the handler lacks a business outcome, and the production runbook must give that review a deadline compatible with the fulfillment SLO.

Keep classification separate from grouping. A grouping rule should be tested for both fragmentation, where one enrollment defect becomes many groups because purchase references differ, and collision, where unrelated defects collapse into one group because their messages resemble each other. Preserve the original sanitized exception and stack trace in capture; use an opaque purchase reference for correlation, without putting student names, email addresses, or payment details into messages.

## 2. Run a local noise-policy rehearsal before connecting capture

This standard-library Go program exercises a proposed application policy. It makes no network requests, implements no vendor grouping algorithm, and uses local fixture fields rather than pretending they are an error tracking API schema. Save it as main.go and run `go run main.go`.

The fixtures model a course checkout with an expected rejection, two attempts at the same enrollment failure, an unfamiliar exception, and a second purchase affected by the same defect. They are synthetic. The rehearsal retains each unexpected attempt as evidence while counting only one notification candidate per known affected purchase; that deliberate loss of attempt-level notification detail is acceptable only because investigation evidence remains available separately.

```go
package main

import "fmt"

type attempt struct {
    purchase string
    stage string
    expected bool
    impact string
}

func main() {
    fixtures := []attempt{
        {"course-a", "payment", true, "none"},
        {"course-b", "enrollment", false, "access-missing"},
        {"course-b", "enrollment", false, "access-missing"},
        {"course-c", "checkout", false, "unknown"},
        {"course-d", "enrollment", false, "access-missing"},
    }
    retained, review, notifications := 0, 0, 0
    affected := make(map[string]bool)
    for _, a := range fixtures {
        if a.expected {
            fmt.Printf("expected outcome: %s stage=%s\n", a.purchase, a.stage)
            continue
        }
        retained++
        fmt.Printf("retain exception: %s stage=%s impact=%s\n",
            a.purchase, a.stage, a.impact)
        if a.impact != "access-missing" {
            review++
            continue
        }
        if !affected[a.purchase] {
            affected[a.purchase] = true
            notifications++
        }
    }
    if retained != 4 || review != 1 || notifications != 2 {
        panic("noise policy changed: inspect retained evidence and impact handling")
    }
    fmt.Printf("retained=%d review=%d notification-candidates=%d\n",
        retained, review, notifications)
}
```

The expected result is four retained exceptions, one review item, and two notification candidates. Changing the unfamiliar exception to an expected outcome should fail this check. A production implementation must validate classification at the application boundary; a caller-controlled “expected” flag would let untrusted input decide what operators can see.

This is a policy rehearsal, not a production notifier. Its map is process-local, its fixtures contain no real stacks, and its counters have no persistence. Durable notification deduplication needs a defined time window, recovery behavior, and shared state across workers; capture-write idempotency and purchase fulfillment idempotency remain separate concerns. Do not copy a local demonstration into the on-call path and assume a restart preserves its decisions.

Capacity planning follows the retained stream. Under a hypothetical burst of 200 unexpected failed attempts per minute for 10 minutes, budget for 2,000 capture attempts before transport retries, even if those attempts concern far fewer purchases. Notification deduplication reduces interruption volume; it does not reduce the evidence ingestion requirement. Include bounded buffering and capture timeouts in the production design so telemetry backpressure cannot consume the checkout worker pool.

Connect capture only after the rehearsal passes. Use the chosen service's documented request schema, preserve sanitized stack evidence from the Node.js process, and verify that the returned event can be found again. Transport retries need a stable event identity, checked response statuses, and bounded exponential backoff that honors Retry-After on HTTP 429. Those are integration requirements, not behavior implemented by this local program.

For the live capture check, save this separate program as capture.go. Set BACKEND_BASE_URL to the documented HTTPS v1 base, INFRAI_API_KEY to the service key, CAPTURE_FILE to a sanitized capture JSON file validated against public discovery, and CAPTURE_IDEMPOTENCY_KEY to a stable event identifier. Run `go run capture.go`. The program passes the schema-valid payload through unchanged, so it neither invents capture fields nor fabricates a Node.js stack. Keep the same identifier when repeating an uncertain write; different exception attempts need distinct event identities.

```go
package main

import (
    "bytes"
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
    if value == "" { panic("missing " + name) }
    return value
}

func main() {
    base := strings.TrimRight(required("BACKEND_BASE_URL"), "/")
    parsed, err := url.Parse(base)
    if err != nil || parsed.Scheme != "https" || parsed.Host == "" {
        panic("require an HTTPS service base URL")
    }
    key := required("INFRAI_API_KEY")
    identity := required("CAPTURE_IDEMPOTENCY_KEY")
    body, err := os.ReadFile(required("CAPTURE_FILE"))
    if err != nil { panic(err) }
    if !json.Valid(body) { panic("capture file must be valid JSON") }
    client := &http.Client{
        Timeout: 10 * time.Second,
        CheckRedirect: func(req *http.Request, via []*http.Request) error {
            return http.ErrUseLastResponse
        },
    }
    for attempt := 0; attempt < 4; attempt++ {
        req, err := http.NewRequest("POST", base+"/errors/capture", bytes.NewReader(body))
        if err != nil { panic(err) }
        req.Header.Set("Authorization", "Bearer "+key)
        req.Header.Set("Content-Type", "application/json")
        req.Header.Set("Idempotency-Key", identity)
        resp, err := client.Do(req)
        if err != nil { panic(err) }
        data, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
        resp.Body.Close()
        if readErr != nil { panic(readErr) }
        if resp.StatusCode >= 200 && resp.StatusCode < 300 {
            fmt.Println("Capture accepted; verify the event in search and group detail.")
            return
        }
        if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
            panic(fmt.Sprintf("HTTP %d: %s", resp.StatusCode, data))
        }
        delay := time.Second * time.Duration(1<<attempt)
        retry := resp.Header.Get("Retry-After")
        if seconds, err := strconv.Atoi(retry); err == nil && seconds >= 0 {
            if seconds > 30 { panic("retry deferred: reschedule after Retry-After") }
            if wait := time.Duration(seconds)*time.Second; wait > delay { delay = wait }
        } else if date, err := http.ParseTime(retry); err == nil {
            if wait := time.Until(date); wait > delay { delay = wait }
        }
        if delay > 30*time.Second { panic("retry deferred: reschedule after Retry-After") }
        time.Sleep(delay)
    }
}
```

Four attempts, a ten-second request timeout, and a thirty-second maximum local wait are example safeguards, not service limits. A longer server-requested delay stops the program for later rescheduling instead of retrying early. Don't use the diagnostic error body as a student-facing checkout response.

## 3. Buy investigation depth, or own the remaining operations

Use the same fixtures to compare products. The table describes procurement boundaries and demonstrations to request, rather than measured rankings or claims that every product groups a particular stack identically.

Infrai is a lightweight option for backend capture, grouping, and simple querying, with a single API key and a single bill across backend services. Its breadth is concrete: 295 routes across 20 modules behind a simple REST API, so another capability uses the same integration conventions. No SDK is required: plain HTTP works from any language or runtime. Its public discovery is self-describing, with request and response schemas for validating a sanitized checkout payload before delivery.

| Choice | Buy versus build decision | Required demonstration for this checkout |
| --- | --- | --- |
| Sentry | Evaluate an error monitoring environment when the decision includes investigation beyond a small capture interface. | Start with a backend stack, then demonstrate any browser or trace workflow the team expects to depend on. |
| Bugsnag | Evaluate a dedicated error monitoring candidate when exception investigation is the main job. | Compare separate course-purchase attempts without losing the evidence for their underlying failure; assess its workflow against the team's release process. |
| Rollbar | Evaluate a dedicated error monitoring alternative with grouping quality as the acceptance criterion. | Try grouping collisions and fragmentation, then recover an individual purchase's sanitized exception. |
| Grafana | Evaluate a broader observability approach when an existing investigation environment matters more than a minimal capture integration. | Demonstrate the path from a checkout signal to correlated evidence, and identify who operates the collection pipeline. |
| Lightweight capture API | Buy exception capture, searchable events, and group detail views; own the application classification and notification policy. | Recover the retained fixtures, review unknown impact, and prove that a notification candidate reaches the operator. |

These are different purchasing questions. Sentry is the comparison point for a broader investigation requirement; Bugsnag is worth assessing through the release workflow, Rollbar through the grouping rehearsal, and Grafana through the surrounding telemetry pipeline. The table deliberately makes those claims testable instead of asserting an unsupported winner. Obtain the same evidence from each candidate before choosing an SDK, a hosted service, or a pipeline the platform team must operate itself.

The trade-off is operational ownership. Basic capture is not suitable as the sole tool for browser-heavy debugging or distributed investigations; evaluate Sentry for that broader requirement, or Grafana when correlation belongs in an existing telemetry pipeline. A simple interface limits integration work, but application classification and notification policy still need engineering time.

For the lightweight option, scope acceptance to server-side exceptions and simple queries. Assign polling and notification delivery to an explicitly owned component. Require a different investigation layer for source-map deobfuscation, native crash symbolication, session replay, or distributed span-tree analysis; adding trace_id and span_id to logs provides correlation context, which should not be mistaken for a tracing system. Use a separate heartbeat monitor, such as Healthchecks, for an enrollment job that never starts: an exception stream cannot establish that a scheduled job ran.

Keep personal data out of telemetry by design. Before choosing any shared log store for student information, separately verify deletion, export, and retention requirements; sanitized purchase references are a narrower starting point than copying the checkout request body. The platform roadmap should include the work surrounding capture, because buying ingestion does not assign an on-call owner or define an SLO.

## 4. Verify evidence recovery, then rehearse rollback

First run the local program unchanged and confirm its four retained exceptions, one review item, and two notification candidates. Then change the unknown-impact fixture to expected and confirm that the assertion fails. This negative check matters: a policy that only demonstrates fewer notifications can pass while hiding a genuine enrollment failure.

Next use the separate capture program with a sanitized payload that satisfies the documented schema. Acceptance requires finding the event again through search and group detail, checking that its stack still identifies the failing checkout stage, and checking that unrelated failures remain distinguishable. HTTP acceptance alone is insufficient. Exercise the documented rate-limit response in an isolated test harness before enabling delivery, and verify that the same event identity survives an uncertain retry while separate attempts retain separate identities.

Rehearse the operator path.

In a staging checkout, let the expected rejection remain an ordinary outcome, retain the unfamiliar exception for review, and follow the known access-impact candidate through the separately owned notification component. Compare the time to review against the fulfillment SLO. Measure capture volume and buffer pressure separately from notification count; the hypothetical 2,000-attempt burst remains an ingestion sizing exercise even when purchase-level deduplication produces a much quieter operator queue.

Roll back a noisy classification change by restoring the previous application policy and checking the same fixtures again. Keep the capture transport switch separate from the classification switch, so stopping a faulty outbound integration does not require changing payment or enrollment behavior. If capture must be disabled, record the evidence gap and continue checking the purchase-to-access SLO through its independent business measurement. Do not silence the fulfillment check merely because the error dashboard looks calm.

Choose the smallest investigation surface that passes those checks, with an owner for every remaining operational step. A manageable stream that preserves unknown failures is a stronger result than a tidy dashboard that cannot explain a missing course enrollment.

## References

- OpenTelemetry, Logs signal concepts: https://opentelemetry.io/docs/concepts/signals/logs/
- Martin Fowler, Feature Toggles: https://martinfowler.com/articles/feature-toggles.html
- Sentry documentation: https://docs.sentry.io/
- Bugsnag documentation: https://docs.bugsnag.com/
- Rollbar documentation: https://docs.rollbar.com/
- Grafana documentation: https://grafana.com/docs/
- Healthchecks documentation: https://healthchecks.io/docs/
