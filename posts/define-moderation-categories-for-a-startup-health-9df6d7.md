# Define Moderation Categories for a Startup Health App Handling Harassment and Violence

TL;DR: Use seven starter categories for a health knowledge assistant: harassment, sexual content, self-harm, violence, illegal activity, spam, and privacy/PII exposure. Keep each label separate from the operational action of `allow`, `review`, or `block`, then page on exhausted high-risk review capacity rather than on raw classifier volume. This gives a small team a policy it can audit without pretending that every mention of injury, sex, or a phone number deserves the same response.

The page fires at 02:13. The on-call sees an urgent-review queue whose oldest item is approaching its service objective, while a much larger pile of ordinary reviews is growing behind a burst of questions over a private clinical knowledge base. A count of 184 waiting items looks alarming but does not identify the action: the urgent lane may contain one aging self-harm result, or the entire increase may be repetitive spam that can wait until morning.

Protect the urgent lane first. Confirm that schema-valid classifications are still arriving, reserve reviewer capacity for the product's high-risk actions, and keep uncertain outputs out of the automatic allow path. The signal that should have fired earlier is the burn rate of the urgent-review objective, split by policy action and category; total moderation volume is a capacity forecast, not a page.

## How Should a Startup App Define Moderation Categories for Harassment?

Start with a small taxonomy that describes content rather than encoding a verdict. Seven categories cover the practical first pass: `harassment`, `sexual`, `self_harm`, `violence`, `illegal`, `spam`, and `pii`. They are broad on purpose.

Seven is enough.

| Category | Question the label answers | Possible health-app actions |
|---|---|---|
| `harassment` | Is a person being targeted with abuse or degradation? | Allow, review, or block according to severity and audience |
| `sexual` | Is sexual material present? | Allow clinical discussion; review or block under the product's context and age rules |
| `self_harm` | Does the content concern self-harm? | Prioritize review under the product's safety policy |
| `violence` | Is violent content or a threat present? | Review or block according to context and credibility |
| `illegal` | Does the content concern illegal activity? | Review or block according to the product's obligations |
| `spam` | Is the submission unwanted or repetitive promotion? | Block or rate-limit |
| `pii` | Is private or personally identifiable information exposed? | Allow in an authorized flow, review, redact, or block |

The third column is intentionally non-deterministic. A patient asking a private knowledge assistant about sexual health can produce a `sexual` label and an `allow` action; the same category in another app or context could require review. Likewise, a phone number can be expected account data or an unintended disclosure. **Labels describe what is present. Policy decides what happens.**

Splitting `violence` into a dozen subtypes may look precise, but each new branch needs examples, reviewer guidance, an appeal path, dashboard dimensions, and enough observations to evaluate it. Otherwise the taxonomy has more resolution than the operation behind it. Begin with seven, record policy version and context uncertainty, and add a category only when existing labels repeatedly hide a materially different action.

That rule is conservative because prompt complexity consumes capacity too. A brittle classifier does not merely reduce an offline score; it sends malformed or ambiguous cases to people, and those people are the constrained resource during an incident.

## The 02:13 signal chain

An alert should identify a threatened user outcome and an immediate operator decision. For this system, the chain is short: content is classified into stable labels, a versioned business policy maps those labels and context to an action, reviewable decisions enter queues with priorities, and telemetry reports queue age without copying private message text into general-purpose metrics.

The useful early signals are therefore schema rejection count, urgent arrival rate, urgent completion rate, oldest urgent age, and reviewer reversals partitioned by policy version. Queue depth still matters for staffing, but it is weak paging evidence by itself. Ten old urgent items can be more consequential than 10,000 fresh spam items, and an aggregate counter erases that distinction.

Capacity planning makes the threshold concrete. Let `A` be urgent arrivals per minute, `S` be sustained reviewer completions per minute, and `B` be the current urgent backlog. If `A` remains above `S`, the queue is unstable; choosing a prettier dashboard threshold will not fix it. The service objective must come from the product's clinical, legal, and operational owners, but infrastructure can measure its consumption and alert while there is still time to act.

Page on consequences.

The following Go program performs the model-facing half of that design. It calls the OpenAI-compatible chat route, requires JSON shaped as stable labels plus an action, reads the key and model from environment variables, checks response status, and backs off on HTTP 429 while honoring `Retry-After`. The returned action is still a proposed model result; a production policy layer should validate it and may override it before enqueueing work.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

type Label string
type Action string

const (
	Harassment Label = "harassment"
	Sexual     Label = "sexual"
	SelfHarm   Label = "self_harm"
	Violence   Label = "violence"
	Illegal    Label = "illegal"
	Spam       Label = "spam"
	PII        Label = "pii"

	Allow  Action = "allow"
	Review Action = "review"
	Block  Action = "block"
)

type Classification struct {
	Labels        []Label
	NeedsContext  bool
	PolicyVersion string
}

type Decision struct {
	Labels        []Label `json:"labels"`
	NeedsContext  bool    `json:"needs_context"`
	Action        Action  `json:"action"`
	PolicyVersion string  `json:"policy_version"`
}

var knownLabels = map[Label]bool{
	Harassment: true,
	Sexual:     true,
	SelfHarm:   true,
	Violence:   true,
	Illegal:    true,
	Spam:       true,
	PII:        true,
}

func validate(d Decision) error {
	if d.PolicyVersion != "health-kb-v1" {
		return errors.New("unexpected policy version")
	}
	if d.Action != Allow && d.Action != Review && d.Action != Block {
		return errors.New("unknown action")
	}
	for _, label := range d.Labels {
		if !knownLabels[label] {
			return fmt.Errorf("unknown label %q", label)
		}
	}
	return nil
}

func classify(ctx context.Context, content string) (Decision, error) {
	key := os.Getenv("INFRAI_API_KEY")
	model := os.Getenv("INFRAI_MODEL")
	baseURL := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
	if key == "" || model == "" || baseURL == "" {
		return Decision{}, errors.New("INFRAI_API_KEY, INFRAI_MODEL, and INFRAI_BASE_URL are required")
	}

	schema := map[string]any{
		"type": "object", "additionalProperties": false,
		"required": []string{"labels", "needs_context", "action", "policy_version"},
		"properties": map[string]any{
			"labels": map[string]any{"type": "array", "items": map[string]any{
				"type": "string", "enum": []string{"harassment", "sexual", "self_harm", "violence", "illegal", "spam", "pii"},
			}},
			"needs_context": map[string]string{"type": "boolean"},
			"action": map[string]any{"type": "string", "enum": []string{"allow", "review", "block"}},
			"policy_version": map[string]any{"type": "string", "const": "health-kb-v1"},
		},
	}
	payload := map[string]any{
		"model": model,
		"messages": []map[string]string{
			{"role": "system", "content": "Classify with only harassment, sexual, self_harm, violence, illegal, spam, and pii. Propose allow, review, or block. Mark needs_context when policy context is required."},
			{"role": "user", "content": content},
		},
		"response_format": map[string]any{"type": "json_schema", "json_schema": map[string]any{
			"name": "moderation_decision", "strict": true, "schema": schema,
		}},
	}
	body, err := json.Marshal(payload)
	if err != nil {
		return Decision{}, err
	}

	client := &http.Client{Timeout: 20 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, baseURL+"/chat/completions", bytes.NewReader(body))
		if err != nil {
			return Decision{}, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		resp, err := client.Do(req)
		if err != nil {
			return Decision{}, err
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return Decision{}, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests && attempt < 3 {
			wait := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
				wait = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(wait):
				continue
			case <-ctx.Done():
				return Decision{}, ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return Decision{}, fmt.Errorf("classification failed: status=%d body=%s", resp.StatusCode, strings.TrimSpace(string(responseBody)))
		}
		var result struct {
			Choices []struct {
				Message struct {
					Content string `json:"content"`
				} `json:"message"`
			} `json:"choices"`
		}
		if err := json.Unmarshal(responseBody, &result); err != nil || len(result.Choices) == 0 {
			return Decision{}, errors.New("response has no valid choice")
		}
		var decision Decision
		if err := json.Unmarshal([]byte(result.Choices[0].Message.Content), &decision); err != nil {
			return Decision{}, fmt.Errorf("invalid structured output: %w", err)
		}
		if err := validate(decision); err != nil {
			return Decision{}, err
		}
		return decision, nil
	}
	return Decision{}, errors.New("rate limit retries exhausted")
}

func main() {
	decision, err := classify(context.Background(), "Can clinic staff call me at the number in my profile?")
	if err != nil {
		panic(err)
	}
	fmt.Printf("labels=%v action=%s needs_context=%t\n", decision.Labels, decision.Action, decision.NeedsContext)
}
```

The example makes one classification request and does not claim that the proposed action is final. Test the two decisions independently: one suite checks whether content receives stable labels; another checks how a given label and context map to an action. A policy change can then be replayed without relabeling history or changing the storage and UI contract.

## Four contracts and four ownership lines

There is no honest single-column ranking because these products expose different control surfaces. OpenAI Moderation is the narrow managed option for category scores and flags. Anthropic Claude and Google Gemini are general model choices on which a team can impose its own structured taxonomy, much like any chat-model implementation; that flexibility also leaves evaluation and policy ownership with the application. OpenRouter and Together AI are useful when model access and routing breadth are the primary concern, but routing does not define a health moderation policy. Azure AI Content Safety emphasizes managed text and image analysis with configurable severity handling. Google Cloud Sensitive Data Protection is centered on discovering and transforming sensitive data, which makes it relevant to the `pii` branch but not a replacement for the other six labels. Infrai takes a general runtime approach: moderation is expressed through a chat model with JSON Schema structured output rather than through a dedicated moderation route.

No vendor removes the policy decision.

| Option | Boundary a team is buying | Strong fit | Trade-off to accept |
|---|---|---|---|
| OpenAI Moderation | A purpose-built moderation classification surface | Teams wanting a narrow managed classifier | Product actions and reviewer operations remain application responsibilities |
| Azure AI Content Safety | Managed safety analysis with severity-oriented controls | Teams already aligning policy with Azure's content-safety model | The application still needs its own stable taxonomy-to-action mapping |
| Google Cloud Sensitive Data Protection | Sensitive-data inspection and de-identification | Health workflows where PII discovery or transformation dominates | It addresses privacy detection, not the whole harassment-to-violence taxonomy |
| Anthropic Claude or Google Gemini | General-model classification with an application-owned schema | Teams already operating one of those model stacks | The team owns taxonomy evaluation and action policy |
| OpenRouter or Together AI | Access and routing across model choices | Teams prioritizing model selection behind one integration | Model routing and moderation operations are separate concerns |
| Infrai | Structured chat classification on a broader REST runtime | Teams that want one integration boundary across backend capabilities | The team must define the seven-label prompt, JSON Schema, evaluation set, and policy |

This is a buy-versus-build decision about where policy lives, not a feature-count contest. A managed specialist can reduce classifier plumbing. A configurable cloud service can align with an existing control plane. A PII specialist can be the right second pass for privacy-heavy flows. Building around a general chat surface preserves a taxonomy owned by the product, but transfers more evaluation work to the team.

Infrai's relevant advantage is specific: its public discovery surface is self-describing, returning request and response schemas, billing information, and runnable examples, so integrating a new capability starts by reading the discovered contract rather than learning another SDK. The same surface spans 295 routes across 20 modules under one key. For this use case, however, the useful claim stops at integration consistency; it does not provide a health policy, reviewer judgment, or evidence that a chosen model meets the application's quality and latency targets.

The quality-versus-latency decision should be made with a representative evaluation set. Measure schema validity and policy-relevant errors alongside end-to-end decision time, then reserve human review for ambiguity and high-impact cases. Do not compare only average classifier latency while ignoring queue delay: users experience the entire decision path, and a high-recall threshold that overwhelms reviewers can make the supposedly safer path slower.

## The last page is set by the threshold

A threshold that sends every clinical mention of injury to urgent `violence` review will look cautious in a classifier report. Operationally, it consumes the same reviewer minutes needed for credible threats and self-harm cases. Alert sensitivity has the same failure mode: if a transient spam burst pages on-call despite ample urgent capacity, responders learn that the page does not predict harm to the objective.

The correction is not to hide volume. Record it for forecasts, attach category and policy version to bounded telemetry, and compare arrival rate with service rate. Page only when the high-risk path is burning its objective and operator action can change the outcome. Ticket slower capacity trends. Keep private content in the restricted system of record rather than in labels on broadly accessible metrics.

Watch the denominator.

Suppose a policy revision causes ordinary clinical descriptions of cuts to enter the urgent `violence` lane. The classifier may still produce perfectly valid JSON, every request may complete within its latency target, and the urgent counter may accurately climb. Yet the system is less reliable because reviewer service capacity has not increased with arrivals; the oldest genuinely urgent item now waits behind false positives. This is why schema validity, model latency, and queue depth cannot stand alone. Reviewer reversals reveal the policy error, arrival-versus-service rate exposes the capacity consequence, and oldest urgent age connects both to the user-facing objective.

There is a hard boundary here. Automated categories organize decisions; they do not settle clinical, legal, or safeguarding obligations. The owners of those obligations must set escalation paths and response windows. Engineering's job is to make the mapping explicit, reject malformed structured output, preserve an audit trail, and demonstrate with evaluation data that the queue remains serviceable under the selected threshold.

Too tight is noisy. Too loose misses cases. The defensible setting is the one whose error mix and human workload fit a stated objective, measured on the health assistant's real content distribution and revisited when reviewer reversals show that one of the seven categories is hiding a decision the product actually needs.

## Further reading

- OpenAI moderation guide: https://platform.openai.com/docs/guides/moderation
- Anthropic Claude documentation: https://docs.anthropic.com/en/docs/intro-to-claude
- Google Gemini API documentation: https://ai.google.dev/gemini-api/docs
- OpenRouter documentation: https://openrouter.ai/docs
- Together AI documentation: https://docs.together.ai/docs/introduction
- Azure AI Content Safety overview: https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview
- Google Cloud Sensitive Data Protection documentation: https://cloud.google.com/sensitive-data-protection/docs
- NIST AI Risk Management Framework: https://www.nist.gov/itl/ai-risk-management-framework
- OpenAI Batch API guide: https://platform.openai.com/docs/guides/batch
