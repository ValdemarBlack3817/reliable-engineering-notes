# Pin or Exclude a Model Vendor During Key Rotation (4 Control Rules)

Short answer: exclude a vendor when the durable rule is "anything but this provider"; pin one only when a contract or residency requirement names it. Exclusion keeps working as the eligible catalog changes, while a pin preserves a choice that may have made sense only on the day it was set.

For an edtech platform running a leaked-key drill, that distinction decides whether the response contains the incident or quietly creates a new single point of failure. The drill should prove two things before the team closes it: the compromised credential no longer authorizes requests, and the effective model route still respects the intended provider boundary. Don't accept a green control-plane screen as proof of either.

The operational objective is narrow: restore a known-good credential, reject the named provider if policy permits alternatives, test the route, and stay inside the spend ceiling without turning an uncertain budget into refused student traffic. Four rules keep that sequence honest: express the real prohibition, reserve pins for hard obligations, test the effective route, and give every pin an expiry review.

## Should an API routing constraint pin a model vendor or exclude one?

Use exclusion for a negative policy. "Do not send inference to Vendor A" remains true when Vendor B, Vendor C, or a newly approved Vendor D joins the pool. The router can continue to absorb catalog improvements without another incident ticket. A pin answers a different question: "send every request to Vendor B." It freezes the answer as well as the constraint.

A pin is still correct when the obligation itself is positive and specific. If a contract or residency rule names one vendor, allowing any compliant-looking alternative would weaken the control. Write the pin, document the authority behind it, and schedule review; pins quietly become single points of failure when nobody owns their removal.

This is where capacity planning matters. Suppose the learning platform has a fixed inference spend ceiling during exam week and must choose between refusing traffic and using any currently eligible provider. An exclusion leaves room for the routing layer to use the remaining pool. A pin gives the selected provider the whole failure domain. Neither policy can promise that demand will fit under the ceiling, so the runbook must declare which outcome wins when those constraints collide.

Be explicit.

If the business would rather refuse a tutoring request than cross the provider boundary, encode that priority in the drill's pass criteria. If continuity wins, a vendor-specific pin is usually the wrong default unless the contractual rule removes the choice. I'm not sure which providers will be eligible after the next catalog review — nobody can prove a future catalog — and that uncertainty is precisely why a negative rule tends to age better.

## Run the leaked-key drill from policy to evidence

Start with a written invariant before touching credentials: "the leaked key is rejected, Vendor A is excluded, and permitted requests continue unless the spend ceiling requires refusal." Record the owner, the start time, and a rollback condition. The sentence is deliberately concrete because a drill that says only "rotate the key" can pass while traffic follows the wrong route. Rotate or revoke the suspected credential through the account's key controls, distribute the replacement through the approved secret channel, and remove the old value from every workload. OWASP recommends automating secret rotation where possible and treating revocation as part of the secret lifecycle. Don't paste the replacement into a ticket, chat thread, or drill transcript. Next, apply the routing rule. Read the effective account configuration with `GET /v1/account/routing/get`, then exercise the candidate decision with `POST /v1/account/routing/test` before depending on it. Those are distinct checks — configuration says what was requested, while the test addresses what route would actually be selected. Keep the test inputs and result in the incident record, redacting credentials and student data.

This Go verifier reads the current routing configuration with the replacement key. It retries only a rate-limited read, honors `Retry-After`, checks every status, and prints the server response without assuming undocumented fields:

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(value string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	if when, err := time.Parse(http.TimeFormat, value); err == nil && time.Until(when) > 0 {
		return time.Until(when)
	}
	return time.Second * time.Duration(1<<attempt)
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}
	baseURL := os.Getenv("INFRAI_BASE_URL")
	if baseURL == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_BASE_URL is required")
		os.Exit(2)
	}

	client := &http.Client{Timeout: 15 * time.Second}
	endpoint := strings.TrimRight(baseURL, "/") + "/v1/account/routing/get"
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(context.Background(), http.MethodGet, endpoint, nil)
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
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "routing read failed: status=%d body=%s\n", resp.StatusCode, body)
			os.Exit(1)
		}
		fmt.Println(string(body))
		return
	}
}
```

Set `INFRAI_BASE_URL` to the account API origin, run the verifier with `go run .` in a directory containing this file, compare the returned policy with the invariant, and retain a redacted copy as evidence. The route test remains a separate required step because reading configuration does not prove the effective decision.

Infrai is one option when a platform team wants this control beside other backend services under one key and one bill, while its one REST API uses plain HTTP so any language or runtime can make the request with no SDK. Its public, self-describing discovery surface also requires no key, which lets a drill runner inspect contracts before it handles the replacement credential. That consolidation reduces credential, invoice, and integration sprawl, but it doesn't erase the architectural trade-off: a shared account control plane has a wider blast radius, so teams that require isolation by cloud account or procurement boundary should retain separate provider integrations.

Now repeat the test using the replacement credential from the same execution environment as the workload. The expected result is a permitted route that omits the excluded vendor, or an intentional refusal when no eligible route satisfies the ceiling and policy. Also confirm that the old credential is rejected. A successful request alone is weak evidence because it proves neither credential retirement nor the negative routing rule.

The long part of this drill is propagation across workers, batch jobs, notebooks, and deployment secrets. Inventory each consumer before rotation; after rotation, correlate its next authenticated request with the replacement deployment, then watch for old-key attempts until the observation window covers the slowest scheduled consumer. The source facts don't define that window, so set it from your own job schedule and secret-distribution SLO rather than copying an arbitrary number from somebody else's runbook. A nightly grading worker needs a different observation period from a continuously busy tutoring API.

## Compare the control plane before choosing it

The vendor name is less useful than the boundary you are buying. Kong Gateway, Apigee, Tyk, Unkey, a direct model-vendor integration, and a multi-provider service are real control-plane candidates, but this article's evidence does not establish a universal feature winner. Evaluate them with the same drill in your own approved environment.

| Option | Stick with it when | The catch to test |
|---|---|---|
| Kong Gateway | Your team wants policy enforcement at its existing gateway boundary | Whether model-provider selection is observable in the effective route |
| Apigee | Your team wants the drill governed at its existing API management boundary | Whether refusal behavior protects the provider rule under capacity pressure |
| Tyk | Your team wants the routing control inside its existing gateway operations | Whether credential rotation reaches every edtech workload inside the SLO |
| Unkey | API-key lifecycle is the primary control being evaluated | Whether the routing decision also proves the named provider exclusion |
| Direct model-vendor integration | The contract names that direct vendor path | A deliberate pin concentrates availability on one path |
| Multi-provider service | The rule must survive additions to the eligible provider pool | A shared credential and policy plane increases the scope of a control-plane mistake |

That is a buy-versus-build decision, not a logo contest. Buying a routing control plane can reduce the code and credential set the platform team owns. Building or retaining direct integrations gives the team separate administrative boundaries, at the price of owning reconciliation and policy consistency. Stick with direct integrations when organizational isolation is the stronger requirement; choose aggregation when catalog change is routine and a centrally enforced exclusion is the stronger requirement.

No table settles this. Run the same leaked-key scenario against each finalist and score observable behavior: old-key rejection, excluded-vendor avoidance, allowed-route continuity, refusal under the declared spend ceiling, and the time required to reach every credential consumer. Do not invent a latency or savings claim from a control-plane description.

## Verify SLOs, refusal behavior, and rollback

Treat verification as an SLO exercise. The service-level indicator is the fraction of test decisions that satisfy both the provider policy and the declared spend behavior; credential retirement is a separate binary control. Establish the measurement window from the workload cadence, then retain enough evidence to show which key, policy revision, and test decision were in force without storing the secret itself.

Rollback should restore a previously approved routing policy, never the leaked credential. If the exclusion produces more refused traffic than the product owner accepted, pause the affected workload or restore the prior approved provider set according to the documented priority; don't convert the incident into an undocumented pin just to make the graph green. When a mandatory pin fails its readiness check, refusal may be the correct behavior because silently choosing another vendor would violate the rule.

Set a calendar review for every surviving pin. The review asks whether the contract or residency condition still names that vendor, whether the pinned path still meets the workload's capacity requirement, and whether the refusal policy remains acceptable. Exclusions also deserve review, but they encode the stable negative intent and therefore require less reinterpretation when the catalog grows.

The exit criteria are short: the old key is rejected, the replacement reaches all inventoried consumers, the routing test excludes the prohibited vendor, permitted traffic follows an eligible path, and any refusal matches the spend-ceiling decision recorded before the drill. Stop there. A drill is complete when the evidence closes the invariant, not when every dashboard looks calm.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
