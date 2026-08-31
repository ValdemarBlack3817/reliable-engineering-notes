# GDPR Backup Storage for User Uploads and Database Dumps in Private European Buckets

Short answer: for a European marketplace that must retain signed documents until an explicit deletion deadline, make the deletion ledger and restore SLO the primary controls, with private object storage as one part of the design. User uploads and database dumps belong in the recovery plan; they should not determine how signed documents are deleted.

The distinction matters. A database dump can preserve a document record after the object has been deleted, while an object lifecycle rule can delete a file without proving that every derived copy is gone. I would therefore model retention as an application decision with an auditable deadline, then use object storage for durable bytes and a separate job for enforcement. Storage is the delivery mechanism. The ledger is the policy.

## The incident is a valid document surviving its deadline

The production scenario I design around is bounded: a buyer and seller sign a marketplace agreement, the application stores the PDF and its metadata, and the contract reaches its deletion date while a nightly database dump still contains the PDF or its object key. Nothing has failed from the storage service's point of view. The compliance failure is in the system boundary.

Deletion has to cover the object, active metadata, thumbnails, temporary exports, and backup copies according to the retention policy. A restore job must not quietly reintroduce an expired document as active data. That is why the recovery runbook needs two stages: restore to an isolated environment, then apply the deletion ledger before any restored data becomes production-visible.

This is the invariant: every retained document has a stable identifier, a deletion deadline, an owner for the decision, and a record of each enforcement attempt. Do not derive the deadline from a filename or trust a bucket listing to be the system of record. Listings are useful for reconciliation; they are not a legal or operational ledger.

Short paragraphs help here.

The failure budget is also easy to state. The deletion worker may retry transient storage or database operations, but it must be idempotent, emit a durable result, and alert when the deadline passes without a confirmed deletion. A successful `DeleteObject`-style request is evidence about one object, not proof that a database dump, replica, cache, or export no longer contains the bytes. Your mileage may vary by retention policy and backup medium, but the proof obligation does not disappear.

## How should a European marketplace handle GDPR retention, private buckets, and database dumps?

Start with a data inventory, not a storage quote. For each signed document, record the purpose, legal retention period, deletion deadline, geographic constraint, encryption ownership, and every copy-producing process. The same exercise should identify user uploads that are not signed documents and database dumps that contain both classes of data. Their access paths and deletion guarantees are different. In a real review, I would trace one document ID through the upload handler, preview generator, asynchronous export, primary database, backup job, restore catalog, and support tooling; this is the part that takes longer than creating a bucket, because the unlisted copy is usually owned by a different team and has a different retention assumption. Until that trace is complete, a deletion deadline is an aspiration rather than an SLO.

Use a private bucket with least-privilege credentials. The application can write a document, the retention worker can delete it, and the restore operator can read into an isolated account; none of those identities needs the others' full permissions. Keep object keys opaque, avoid personal data in prefixes, and make audit events identify the document ID without copying the document contents into logs. S3 compatibility is valuable only if the client uses documented common operations and the target's policy, lifecycle, encryption, and region behavior have been tested.

The database dump is the awkward copy. Encrypt it before upload, limit who can restore it, and assign it a retention class that the policy actually permits. If a dump must remain for disaster recovery, the deletion service needs a way to mark the document as expired and remove it after restore, or the organization needs a documented exception with an independent deadline. Saying “the bucket is private” does not answer that question.

| Decision | Managed object storage | Self-hosted storage |
|---|---|---|
| Access control | Validate bucket policy, workload identity, audit events, and delete permissions | Build identity integration, policy enforcement, and audit retention |
| Deadline enforcement | Validate lifecycle granularity and delete visibility; keep the ledger in the application | Operate the same controls plus disks, upgrades, and failure domains |
| Recovery | Measure download path, isolation, and restore time | Operate capacity, repair, replication, and the recovery path |
| Lock-in | Test export, metadata portability, and S3-compatible operations | Own migration tooling and the format of every stored copy |

This is a buy-vs-build gate, not a winner's podium. A small team should count on-call work and capacity headroom as part of the architecture, while a larger team may accept more operational ownership for a control it genuinely needs.

## Prevent deletion races with a ledger and an idempotent Go worker

The worker below shows the control boundary rather than pretending that one SDK call solves retention. A transaction or queue must claim the due row, and the storage adapter must make repeated deletion safe. The important part is that a confirmed object deletion is followed by a ledger update, while an uncertain result remains retryable instead of being marked complete.

```go
package retention

import (
	"context"
	"errors"
	"time"
)

type Document struct {
	ID           string
	ObjectKey    string
	DeleteAfter  time.Time
	DeletedAt    *time.Time
}

type Ledger interface {
	ClaimDue(ctx context.Context, now time.Time) (Document, error)
	MarkDeleted(ctx context.Context, id string, at time.Time) error
	MarkRetryable(ctx context.Context, id string, reason string) error
}

type PrivateStore interface {
	Delete(ctx context.Context, key string) error
}

func EnforceOne(ctx context.Context, ledger Ledger, store PrivateStore, now time.Time) error {
	doc, err := ledger.ClaimDue(ctx, now)
	if err != nil {
		return err
	}

	if err := store.Delete(ctx, doc.ObjectKey); err != nil {
		if markErr := ledger.MarkRetryable(ctx, doc.ID, err.Error()); markErr != nil {
			return errors.Join(err, markErr)
		}
		return err
	}

	return ledger.MarkDeleted(ctx, doc.ID, now.UTC())
}
```

The real test is a crash between the delete request and `MarkDeleted`. Re-running must not make the operation dangerous. Build a reconciliation job that compares ledger state with object inventory, and treat unexplained objects as incidents requiring review, not as permission to bulk-delete. That discipline also protects a marketplace from deleting a document whose deadline was extended by a legitimate dispute process.

## What should a backup comparison measure beyond storage price?

The cheapest backup storage is not identifiable from retained gigabytes alone. Measure the bytes written, retained bytes, request volume, restore-test downloads, worst-case egress, encryption processing, and operator time. Then measure the thing the SLO actually promises: how long it takes to recover a usable application state while applying the deletion ledger.

For each candidate, run the same fixture: signed PDFs, ordinary user uploads, a database dump, a manifest, and an intentionally expired document. Test private access, wrong-identity denial, concurrent writers, retry behavior, export, and restore into isolation. Record the result with the region and object sizes. I’m not sure which storage option will be cheapest for an unknown workload; a confident ranking without those inputs is a spreadsheet artifact.

The catch is that this design is not suitable when the business needs public media delivery, millisecond object mutation, built-in legal hold, or automatic multi-region recovery that the selected storage configuration cannot provide. Choose a service with those controls, or separate that workload from the private backup bucket. Stick with a surrounding cloud's native identity and audit path when reducing access-control toil is more important than a portable endpoint. Choose self-hosting only when its operational burden is an explicit, funded requirement.

Do the restore drill quarterly or at the interval set by the recovery policy. A green upload metric is not a restore SLO. The only result that matters during an incident is a readable recovery point, a known deletion outcome, and a record someone can defend later.

## References

- https://developers.cloudflare.com/r2/
- https://cloud.google.com/storage/docs
