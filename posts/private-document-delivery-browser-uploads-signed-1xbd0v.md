# Private Document Delivery: Browser Uploads, Signed URLs, Node.js, and Postgres

Short answer: have the browser upload report bytes directly to private object storage with a short-lived signed upload URL, keep authorization and workflow state in Postgres, and issue a signed download URL only after the application has checked the customer's access and the object's ready state.

For a customer-support SaaS, delivery simplicity is tempting: generate a link and let the browser fetch the file. The access-control boundary is the part that deserves the design time. A generated report can contain a ticket transcript, account identifiers, or internal notes, so the storage key cannot be the permission model and a successful browser transfer cannot be the application record of completion.

## What should a Node.js service record before private document upload?

Start with an application record in Postgres. It should bind a report to a tenant, a customer, and a report revision, then reserve a fresh object key that the client never gets to choose. Store the original filename and an expected content type for display and validation, but treat them as metadata rather than authority. The authorization decision should come from the authenticated session and the database row.

Use an explicit lifecycle such as `pending`, `ready`, and `deleted`. `pending` means the application granted permission to write a particular key; it does not mean that bytes exist. `ready` means the server verified the object and the report can be offered for download. `deleted` means the product no longer exposes it, even if retention policy keeps the row or bytes for a while.

That distinction prevents a common support incident: the UI says “report ready” because the browser got a successful response, while the process that records completion never ran. The opposite case matters too. A stale object must not become downloadable merely because a user can guess a path or because a storage listing contains it.

Keep every revision under a new key. A mutable key such as `tenant-42/latest.pdf` makes rollback and audit needlessly vague; an immutable revision key lets Postgres point the customer to the intended object. The database transaction can reserve the row and any quota accounting, but it cannot make the later object upload atomic. Design the pending interval and its cleanup as a normal part of the system.

Keep it private.

## How do browser uploads, signed URLs, Node.js, and Postgres fit together?

The write path has four meaningful steps. The authenticated Node.js endpoint validates the tenant and report request, inserts a pending row, and asks the storage adapter for a short-lived signed write URL. The browser sends the bytes directly to that URL. It then calls a finalize endpoint. Finally, the server checks the exact object key and conditionally promotes the row to `ready`.

The download path is shorter but stricter: authenticate the customer, load the report row, check tenant and report permissions, require `ready`, and create a new short-lived signed read URL. The browser follows that URL directly. A signed URL grants narrow transfer authority for a limited time; it does not replace application authorization, and it should never be accepted as proof that a customer may read an arbitrary object.

Here is the part worth making boring. The adapter hides whichever object-storage client the team selects, while the service owns the security decision and the state transition.

```go
package reports

import (
	"context"
	"database/sql"
	"errors"
)

var ErrNotReady = errors.New("report is not ready")

type ObjectStore interface {
	CreateUploadURL(ctx context.Context, key string, contentType string) (string, error)
	ObjectExists(ctx context.Context, key string) (bool, error)
	CreateDownloadURL(ctx context.Context, key string) (string, error)
}

func FinalizeReport(
	ctx context.Context,
	db *sql.DB,
	store ObjectStore,
	reportID string,
	objectKey string,
) error {
	exists, err := store.ObjectExists(ctx, objectKey)
	if err != nil {
		return err
	}
	if !exists {
		return ErrNotReady
	}

	result, err := db.ExecContext(ctx, `
		UPDATE reports
		SET status = 'ready'
		WHERE id = $1
		  AND object_key = $2
		  AND status = 'pending'
	`, reportID, objectKey)
	if err != nil {
		return err
	}

	changed, err := result.RowsAffected()
	if err != nil {
		return err
	}
	if changed != 1 {
		return ErrNotReady
	}
	return nil
}
```

The conditional update is an idempotency boundary. Two finalize requests may arrive, but only one can move the row out of `pending`. The handler should map a repeated finalize to a deliberate conflict or already-complete response; it should not create another report revision just because a browser retried.

## What failure modes should the private document runbook cover?

Test the unhappy path before testing throughput. A customer from tenant A must not obtain a download URL for tenant B. A `pending` row must not produce a download URL. A browser tab closed during upload must leave evidence that a reconciler can inspect. A retry after a timeout must not overwrite a previous report or erase the database's record of which revision was current. Consider a support agent who requests a new report while the previous one is still being generated: the first request reserves revision 17, the browser starts sending it, and the agent refreshes the page before finalization. If the refresh creates a second row pointing at the same human-readable filename, an eventual callback can mark the wrong row ready; if it reuses the storage key, a late upload can replace bytes that an already-issued link was meant to retrieve. A fresh revision key plus the conditional Postgres update makes both callbacks visible and leaves the application with one deliberate current-revision pointer.

The browser's progress event is useful product feedback, not completion evidence. With `XMLHttpRequest`, upload progress can update a report screen while the server still treats the object as pending. Keep the final object check even when the progress bar reaches 100 percent.

The progress bar is not an authorization check.

Track signals that describe the control plane, not only storage bytes:

| Signal | Why it matters | Response |
|---|---|---|
| Age of the oldest pending row | Shows abandoned or slow uploads | Inspect by bounded batches; expire only by policy |
| Ready rows whose objects are absent | Reveals a broken state transition or retention mistake | Stop issuing links for affected rows and reconcile |
| Cross-tenant authorization denials | Shows attack pressure and permission regressions | Alert on a meaningful rate, then inspect request context |
| Finalization latency and mismatch count | Separates browser delivery problems from application problems | Compare by tenant, report size, and network class |

Give the upload and finalize endpoints separate SLOs. The upload URL endpoint should be fast because it returns authorization metadata, while finalization depends on an object check and database write. A `429` should honor `Retry-After` with bounded exponential backoff; an uncontrolled retry loop turns a delivery problem into load on the control plane.

I am not sure what pending timeout fits your customers. A generated support report may be small, but a customer network can still pause for minutes. Measure upload-duration percentiles by file size, add a margin, and make expiry observable. A universal timeout copied from another SaaS is guesswork.

## When is this direct browser delivery pattern the wrong choice?

The pattern is unsuitable when every byte must pass through the application for malware scanning, transformation, legal hold enforcement, or content inspection before storage. In that case, proxying or a separate quarantine stage may be the right operational boundary, even though it increases bandwidth, latency, and application capacity requirements.

It is also a poor fit for permanent public links and public static hosting. Private customer reports need an application authorization check, and signed URLs are intentionally temporary. For immutable archives, verify that the selected storage system provides the retention and versioning controls your compliance policy requires; a Postgres status column is not a substitute for storage-level retention.

The catch is operational ownership. Direct browser transfer reduces application bandwidth, but the team must still configure and test CORS, choose URL lifetimes, clean abandoned uploads, and preserve a useful audit trail. Stick with a provider-native integration when its specific retention, replication, or lifecycle controls are central to the requirement. Use a storage abstraction when portability and a common service contract matter more than exposing every provider-specific feature.

Rollback should be equally explicit. Stop issuing new upload URLs, keep verified `ready` reports readable, and leave `pending` rows for reconciliation. Do not delete objects as part of an application rollback. Once the service is stable, scan old pending rows, check their exact keys, promote only objects that match the expected row, and expire the rest according to the retention policy.

The decision rule is simple: Postgres owns meaning and authorization; private object storage owns bytes; signed URLs make the transfer temporary. That separation keeps a customer-support report easy to deliver without making the delivery link the system's source of truth.

## References

- [AWS S3 documentation: Download and upload objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [MDN: Using XMLHttpRequest](https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest_API/Using_XMLHttpRequest)
