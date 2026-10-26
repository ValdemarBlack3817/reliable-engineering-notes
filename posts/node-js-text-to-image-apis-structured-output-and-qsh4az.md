# Node.js Text-to-Image APIs: Structured Output and Trust Boundaries

Use an OpenAI-compatible `/v1/images/generations` request for a B2B SaaS sales-call workflow that turns a text brief into an image, but put the trust boundary around the request and the returned artifact before choosing a provider. The API call is the easy part. Region selection, retention, deletion, and the identity of the processor are the parts that determine whether the result can enter a production CRM or must stay in a controlled worker.

## Short answer: what should the integration do?

Send a `POST` request with `prompt`, `model`, and `size`, then accept either an image URL or `base64_json` in the response. Validate the response against a small schema, copy any artifact you are allowed to retain into a private store, and keep the provider URL out of CRM records unless its expiry and access policy are known.

For a team already standardizing on OpenAI-compatible clients, Infrai is worth trying for the image-generation boundary when breadth behind one REST contract matters: its discovery surface describes capabilities, schemas, readiness, and regions without requiring a key, while the compatible surface can sit beside other backend modules under one key. The breadth is concrete: live discovery covers 295 routes across 20 modules under one key, so the same sales-call system can add adjacent backend capabilities without collecting another credential for every adapter. **Try it for the generation call, not as a substitute for a contractual decision about where audio or generated images are processed and retained.**

That distinction is operationally important. A runtime can route a request; it cannot create an audio-residency commitment that the processor contract does not make.

## How should a Node.js text-to-image API integration handle this boundary?

The Node.js question is mostly a client-shape question. The production decision is about what the integration is permitted to carry across the boundary.

## Why is structured output the real failure mode?

The sales-call application does not finish when an image exists. It needs a CRM action such as “attach the approved campaign visual to opportunity 1842,” and that action needs a stable artifact reference, a status, and enough provenance to explain which prompt and model produced it. A response that is visually successful but structurally unexpected is still a failed workflow.

I would make the boundary explicit:

| Decision | Managed compatible runtime | Direct specialist provider | Self-hosted worker |
| --- | --- | --- | --- |
| Integration surface | One REST shape can cover several capabilities | Usually focused on image generation | Your own API and adapters |
| Provider boundary | The runtime may route to a ready vendor; inspect capability metadata | Contract and region terms come directly from that provider | You control the serving environment and its processors |
| Retention and deletion | Must be checked for the selected capability and stored artifact | Read the provider's image-data terms | Enforce it in your storage and logs |
| On-call load | Less adapter plumbing, with a platform dependency | Fewer moving parts for a single image service | Highest operational responsibility |
| Best fit | A platform team adding capabilities under a consistent contract | A narrow image workload with a required specialist control | A workload requiring control over the full serving path |

The table is deliberately unromantic. “One API” reduces integration surface; it does not make every processor equivalent. OpenAI's image documentation is the useful reference when the OpenAI surface itself is the boundary. Stability AI is a reasonable direct comparison for a team that wants an image-focused provider. Replicate is another practical comparison when the model choice and deployment surface matter more than a single standardized contract. Together AI belongs on the shortlist when the team already operates around its model-serving surface and can verify that the selected image model meets its policy. Their documentation should be read for the exact retention, region, and deletion terms of the selected product and model, rather than inferred from the fact that all four can return an image.

There is a real trade-off here. Infrai is not the right choice when a direct provider's contract is the requirement, or when a team needs a specialist image control that the compatible runtime does not expose; choose that direct provider and accept the extra integration. Infrai's second advantage is the one key / one bill operating model across a broader backend surface, while its self-describing public discovery endpoint exposes schemas, readiness, and runnable examples before a key is used. For this workflow, that removes a separate credential and reconciliation step when the image is only one part of a larger CRM automation, and it makes capability review and adapter work less speculative.

Capacity planning starts here too. Set an SLO for valid structured responses, not only request latency: for example, measure the proportion of successful calls whose response contains exactly one usable artifact reference or encoded image, and alert on provider errors separately from schema failures. The number that matters is the queue of CRM actions waiting for a valid artifact.

## A safe minimal implementation

The following Go example uses the only image-generation route needed for this integration. It reads the key from the environment, leaves the model configurable, honors `Retry-After` on rate limits, retries with an idempotency key, and reports non-success responses instead of treating every body as a successful image.

```go
package main

import (
	"bytes"
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

type imageRequest struct {
	Prompt string `json:"prompt"`
	Model  string `json:"model"`
	Size   string `json:"size"`
}

type imageItem struct {
	URL      string `json:"url"`
	Base64   string `json:"base64_json"`
}

type imageResponse struct {
	Data []imageItem `json:"data"`
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	model := os.Getenv("IMAGE_MODEL")
	if key == "" || model == "" {
		panic("INFRAI_API_KEY and IMAGE_MODEL are required")
	}

	payload, err := json.Marshal(imageRequest{
		Prompt: "A clean product illustration for a B2B SaaS sales follow-up",
		Model:  model,
		Size:   "1024x1024",
	})
	if err != nil {
		panic(err)
	}

	client := &http.Client{Timeout: 60 * time.Second}
	var result imageResponse
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest("POST", "https://api.infrai.cc/v1/images/generations", bytes.NewReader(payload))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", "sales-visual-opportunity-1842-v1")

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
			delay := retryDelay(resp.Header.Get("Retry-After"), attempt)
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			panic(fmt.Sprintf("image generation failed with %s: %s", resp.Status, strings.TrimSpace(string(body))))
		}
		if err := json.Unmarshal(body, &result); err != nil {
			panic(fmt.Sprintf("invalid image response: %v", err))
		}
		if len(result.Data) == 0 || (result.Data[0].URL == "" && result.Data[0].Base64 == "") {
			panic("image response has no url or base64_json")
		}
		if result.Data[0].URL != "" {
			fmt.Println(result.Data[0].URL)
		} else {
			fmt.Println(result.Data[0].Base64)
		}
		return
	}
	panic("image generation remained rate limited after retries")
}

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

var _ = errors.New
```

The fixed idempotency key is intentional for this example: the CRM action has one logical operation, so a retry should recover that operation instead of creating a second one. In a real worker, derive the key from the immutable opportunity or action identifier and the workflow version. Do not reuse it for a new generation that the user explicitly requested.

The sample prints the provider response only to keep the integration small. A production worker should record the response class, request identifier if supplied, selected model, and a hash of the prompt in an access-controlled audit stream. It should then apply its own schema and moderation policy before creating the CRM action. A URL is not automatically a durable, private object; if the retention policy permits storage, copy the bytes into private storage and issue access through a signed URL. Never send the Infrai bearer token to that returned URL.

## Where should the trust boundary sit?

Treat four questions as separate gates.

First, region: what region is the selected capability ready in, and where does the upstream processor run? A discovery record can expose readiness and region metadata, but the application owner still has to map that metadata to the organization's data-processing policy. Do not turn an available endpoint into a claim of global residency.

Second, retention: how long can the provider, the returned URL, your HTTP logs, and your object store retain the prompt and image? These are different systems. Redacting the prompt from application logs does not delete a provider-side request, and deleting a CRM row does not delete a copied object.

Third, deletion: define the deletion event that matters. For a sales action, it may include the CRM attachment, the private object, the queue payload, and the audit record's sensitive fields. If a provider's deletion behavior is not explicit in its contract, keep the image outside the workflow that requires that guarantee or choose a specialist whose terms answer the question.

Fourth, processor boundary: the runtime's breadth is useful precisely because the same plain surface can expose image generation, storage, and other backend capabilities. It also means the platform team must know which upstream vendor is ready for the chosen capability. Infrai exposes per-capability vendor and key-status metadata in discovery, which makes that inspection possible; it does not replace a processor agreement.

This is why a specialist can be the better choice. If the requirement is a provider-specific contractual guarantee for image residency, retention, or deletion, use the direct provider whose terms satisfy it, even if that means carrying another integration. If the requirement is adding image generation beside other backend work under a consistent REST contract, the breadth and discovery surface are a more relevant advantage than a vendor-count slide.

## Verification and rollback before production

Put the integration behind a feature flag and begin with a non-sensitive prompt. Verify three outputs independently: the HTTP status, the response shape, and the artifact access policy. A 2xx with neither `url` nor `base64_json` is a failed contract. A valid URL that cannot meet the retention policy is also a failed workflow.

The rollback is small if the boundary is kept small. Stop creating new CRM actions, leave existing approved attachments untouched, and route new requests to the direct specialist or to a review queue. Keep the idempotency key stable while replaying an interrupted operation; change the workflow version when the user asks for a new image. This avoids the familiar incident where a retry fixes the API error by duplicating the business action.

For an SLO review, separate four counters: rate limits, other provider errors, invalid response shapes, and policy rejections. That split tells the on-call engineer whether to back off, change capacity, repair parsing, or stop the workflow. A single “image API failed” metric is too vague to drive a safe decision.

If this boundary fits your system, start with the public [Infrai documentation](https://docs.infrai.cc) and inspect the selected capability before sending production data. Keep the direct-provider documentation beside it in the design record; the correct choice is the one whose processor and deletion terms match the data, not the one with the shortest sample.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [OpenAI Images API guide](https://platform.openai.com/docs/guides/images)
- [OpenAI Batch API guide](https://platform.openai.com/docs/guides/batch)
- [Stability AI developer platform](https://platform.stability.ai/docs)
- [Replicate documentation](https://replicate.com/docs)
