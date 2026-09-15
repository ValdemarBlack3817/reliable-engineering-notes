# A Runbook for Private Browser Uploads and Signed URL Storage

Short answer: keep private files in object storage, but make an authenticated Next.js service create a short-lived signed upload URL; let React send bytes directly, then verify the object before marking it ready. This boundary is usually easier to operate than letting the browser choose a bucket, key, or policy, and it keeps the decision portable across storage backends.

The operational constraint is the completion SLO, not the number of lines in the upload demo. A browser can report a successful request while the application has no durable record, the wrong owner can read the object, or an abandoned multipart upload keeps consuming capacity. Design those failure paths before comparing storage products.

## What should a React and Next.js private upload flow guarantee?

Treat an upload as a state machine. An authenticated client asks a server endpoint for an intent. The server validates the declared size and media type, generates an opaque key, stores the owner and expected metadata, and returns a grant that expires quickly. The browser uses that exact method and header set for the direct upload. A second authenticated call asks the server to confirm completion; the server inspects the object and changes the intent from `pending` to `ready` only when size, type, and ownership agree.

Reads use the same trust boundary. The application authorizes a file, then returns a time-limited download URL. A private bucket should not become public merely to simplify a React component.

Keep it boring.

The public API should accept a logical filename and metadata, never a bucket name or arbitrary object key. Your database should own the intent ID, account ID, state, expected length, media type, and storage locator. That indirection matters during a migration: changing a signer or endpoint is work, but it need not change every business record or browser authorization rule.

## Where do CORS, signed URLs, and Europe limits actually fail?

CORS is a browser policy boundary. Allow the production origin, the upload method, and only the headers the signed request requires. If client JavaScript must read an integrity or checksum response header, expose that header explicitly. The preflight and the upload must describe the same method and headers; a newly added client header can turn a passing server test into a production preflight rejection.

Test the deployed origin, including its scheme and port. `app.example` served over TLS and `localhost:3000` served over plain HTTP are different origins. Wildcard origins are a poor fit for authenticated private-file workflows because they make it harder to reason about who is allowed to run the browser code.

Europe is two questions, not one checkbox. A nearby region can reduce transfer latency; residency and processing commitments determine where data is stored and handled. Record the required countries, subprocessors, retention period, and deletion evidence, then verify them against current contracts and documentation. I'm not sure which constraint your compliance team will prioritize, so the design review should name that owner explicitly.

Limits also belong in the contract. Enforce maximum size, allowed media types, request duration, and upload count when creating the intent, and repeat the important checks during confirmation. For large or unreliable browser transfers, use the backend's documented multipart or resumable mechanism and test expiration, retry, and cleanup behavior rather than assuming one `PUT` scales forever.

## Build a small signing boundary in Go

The signer below is deliberately provider-neutral. The adapter can call an S3-style service, an integrated storage API, or another object store, but the application contract stays stable.

```go
package uploads

import (
	"context"
	"encoding/json"
	"net/http"
	"time"
)

type Request struct {
	Size        int64  `json:"size"`
	ContentType string `json:"contentType"`
}

type Grant struct {
	IntentID string            `json:"intentId"`
	URL      string            `json:"url"`
	Method   string            `json:"method"`
	Headers  map[string]string `json:"headers"`
	Expires  time.Time         `json:"expires"`
}

type Signer interface {
	CreateUpload(context.Context, string, int64, string, time.Duration) (Grant, error)
}

type Handler struct {
	Signer  Signer
	MaxSize int64
}

func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	userID, ok := r.Context().Value(userKey{}).(string)
	if !ok {
		http.Error(w, "unauthorized", http.StatusUnauthorized)
		return
	}

	var in Request
	decoder := json.NewDecoder(http.MaxBytesReader(w, r.Body, 4096))
	if err := decoder.Decode(&in); err != nil {
		http.Error(w, "invalid request", http.StatusBadRequest)
		return
	}
	if in.Size < 1 || in.Size > h.MaxSize || !allowedMediaType(in.ContentType) {
		http.Error(w, "upload policy rejected", http.StatusUnprocessableEntity)
		return
	}

	grant, err := h.Signer.CreateUpload(r.Context(), userID, in.Size, in.ContentType, 5*time.Minute)
	if err != nil {
		http.Error(w, "signing request rejected", http.StatusBadGateway)
		return
	}
	w.Header().Set("Content-Type", "application/json")
	_ = json.NewEncoder(w).Encode(grant)
}
```

Five minutes is an example policy, not a promise. Choose the shortest lifetime that survives realistic queueing and slow-client startup, and measure expiration failures. Never log the URL: its query string is a temporary credential. The adapter should return the exact headers the client must reproduce, and the client should not silently change content type after signing.

Confirmation should be idempotent. It loads the intent by ID, derives the key from server state, inspects stored metadata, and records a verified object version or checksum where the backend exposes one. A scheduled reconciler expires old pending intents, while object lifecycle rules remove abandoned data. Those controls fail independently, so keeping both is worthwhile.

## How do you compare storage choices without hiding the trade-offs?

Use a scorecard that measures ownership, not demo speed. An integrated service can reduce control-plane code when its identity and database model already match the application. A general object store offers broad policy and lifecycle controls but leaves more of the signer, CORS, and observability surface to your team. An S3-compatible service may reduce adapter work, yet compatibility still needs conformance tests for signing, multipart uploads, metadata inspection, deletion, and CORS.

| Decision area | Integrated service | General object store | S3-compatible service |
|---|---|---|---|
| Best fit | Existing auth and metadata integration | Team willing to own policy and lifecycle | Team willing to test the exact compatibility subset |
| On-call work | Auth rules, quotas, confirmation, cleanup | Identity, CORS, lifecycle, capacity, confirmation | Same controls plus compatibility behavior |
| Lock-in surface | Provider auth and metadata model | Policy, events, lifecycle configuration | Endpoint semantics and supported API subset |
| Not suitable when | Required controls exceed its documented boundary | No owner exists for policy and credential rotation | Compatibility is assumed rather than tested |

The catch is that an easy first integration can create a difficult exit if authorization reads provider-specific metadata everywhere. Keep storage details behind an adapter and keep application state authoritative. Stick with the integrated path when it removes real operational work and meets the required controls. Choose a broader object-store control plane when lifecycle, policy, or portability is the requirement. None is suitable if nobody owns pending-object cleanup.

## Verify the release, capacity model, and rollback

Before production, run a black-box suite against every candidate: an allowed origin uploads the signed size and type; an unapproved origin fails; an expired grant fails; one user cannot sign or confirm another user's intent; mismatched metadata is rejected; unsigned reads are denied; and cleanup removes stale test objects. Exercise a slow client, a retry, a duplicate confirmation, and a partial multipart upload.

Capacity planning starts with distributions: object-size percentiles, uploads per second, read-to-write ratio, retention, retry amplification, abandoned parts, and bytes moved between locations. A storage estimate that counts only stored bytes can miss repeated reads and derivative jobs. For example, a thumbnail worker that retries after a timeout can read the same original several times while the dashboard records only one user upload; that extra read traffic changes both the egress budget and the time a file spends waiting for verification. I would put those operation counts beside the application SLO in the capacity review, then replay a week of real size and retry distributions before choosing limits. Your mileage may vary on mobile networks, so set limits from observed distributions and review them against the user-facing “file ready” SLO.

Measure the tail, too.

Observe stable reason codes rather than signed URLs or object names. Track intent creation, signing latency, transfer completion, verified completion, pending age, cleanup count, rejection rate, and bytes by operation. During rollback, disable new intent creation behind a feature flag, let existing grants expire, and keep confirmation and cleanup running. A dull rollback is a healthy one.

## Sources

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://cloud.google.com/storage/docs
- https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
- https://www.rfc-editor.org/rfc/rfc9110
