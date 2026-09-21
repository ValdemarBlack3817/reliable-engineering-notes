# Image Crop Quality: 4 Centre and Content-Aware Checks for Upload Moderation

TL;DR: For moderated recipe uploads, use a center crop only when the source already matches the target shape and the important region is known to be central. Otherwise, use a content-aware candidate, but do not trust its score as proof of quality. Compare both candidates against four signals: subject retention, moderation coverage, crop stability, and reviewer overrides. Keep the uncropped upload available to moderation. A clever crop can still hide the very pixels that a reviewer or classifier needed to inspect.

The page arrives as "published image may exclude reviewed content." On-call sees an avatar that looks harmless, opens the original, and finds that the content-aware crop selected a face while excluding a policy-relevant region near the edge. The moderation decision was correct for the derivative it received; the pipeline contract was wrong. The earlier signal should have been disagreement between moderation on the original and moderation on the publishable crop, not a generic image-processing error.

This distinction matters for recipe photos too. A center crop can cut a long platter at both ends, while a saliency-driven crop may lock onto a high-contrast garnish and discard most of the dish. Neither method wins by name. **The useful comparison is the rate at which each method preserves the intended subject and the pixels covered by moderation.**

## What should have paged before the bad crop shipped?

A crop service returning errors is easy to alert on, but successful responses are where quality failures hide. Instrument one record per derivative with the source dimensions, target aspect ratio, chosen crop rectangle, algorithm version, confidence or ambiguity measure, moderation result for the source, moderation result for the derivative, and the final reviewer action. Do not log image bytes or inferred sensitive attributes into ordinary telemetry.

The first actionable signal is a policy disagreement: the source requires review or rejection while the proposed derivative passes. That condition should hold publication for review. It should not page on-call for a single upload, because the system is doing its job; page only when the disagreement rate breaches the service's agreed error budget over a sustained window. The exact threshold has to come from observed traffic and staffing capacity, not a borrowed percentage.

A second signal catches quality degradation without pretending that an automated metric knows what looks good. Track the fraction of crops that a reviewer adjusts or rejects, grouped by target shape and coarse content class such as avatar or dish. Compare a new algorithm version with the current version on the same held-out images before rollout. A release that moves attention from faces to shirt logos, or from a plate to a bright napkin, may have unchanged latency and a worse product outcome.

Four fields are enough to start the trace:

1. `source_moderated`: whether moderation evaluated the original upload.
2. `crop_source_agreement`: whether source and derivative policy outcomes agree.
3. `subject_retained`: a review label for the intended face or dish.
4. `reviewer_changed_crop`: whether a human replaced the proposed rectangle.

No single field is the SLO. Together, they show whether the crop is publishable and whether the moderation boundary still covers what users submitted.

## Should centre or content-aware crop lead image quality for avatars and dishes?

Center cropping is deterministic: calculate a rectangle with the target aspect ratio around the source midpoint, then resize that rectangle. Its operational virtues are real. It is fast, explainable, stable across deployments, and has no model artifact to distribute. For square headshots captured with a centered camera, that predictability is often the right default. It also fails consistently when the subject is off-center, when two people share a frame, or when an upright portrait must become a wide banner.

Content-aware cropping first estimates which region deserves attention, then places the target rectangle around that evidence. The evidence might come from a face detector, object detector, segmentation mask, edge density, or another saliency map. Each choice defines a different meaning of "content." A face detector helps an avatar workflow but says little about a bowl of noodles; edge density may prefer text on packaging over the food beside it. Ambiguous scenes need an explicit fallback rather than an arbitrary winner.

The limitation is symmetric. Centre crop is unsuitable when important subjects routinely sit near an edge; content-aware crop is unsuitable when the available detector does not represent the upload class or when its decision cannot be reproduced during an incident. This trade-off is why avatars and dishes need separate evaluation slices rather than a shared average.

The file format is part of this path. MDN documents that image formats differ in compression behavior, animation support, transparency, and browser support. Decode orientation and animation deliberately, apply policy to the representation the user will see, and encode derivatives in a format supported by the delivery surface. A crop comparison performed on one decoded frame while publication displays another is not a comparison at all.

Here is a deliberately small Go contract. It keeps crop selection separate from moderation and forces callers to retain both assessments; a detector implementation can change without changing the publication rule.

```go
package crop

import "image"

type Proposal struct {
	Rect       image.Rectangle
	Method     string
	Confidence float64
}

type Assessment struct {
	SourceAllowed bool
	CropAllowed   bool
	SubjectHeld   bool
}

type Selector interface {
	Center(bounds image.Rectangle, targetW, targetH int) Proposal
	Aware(img image.Image, targetW, targetH int) (Proposal, error)
}

func Publishable(a Assessment) bool {
	return a.SourceAllowed && a.CropAllowed && a.SubjectHeld
}
```

Notice what `Publishable` does not accept: a saliency confidence score. Confidence can route low-certainty cases to review, but it cannot establish that an unseen edge region was safe.

## Measure coverage before aesthetic preference

Build the evaluation set from consented or appropriately licensed examples that represent the actual upload envelope. Split results by avatars and dishes, then by the conditions that cause cropping trouble: off-center subjects, multiple subjects, extreme source ratios, text near an edge, transparent backgrounds, and animated input if the product accepts it. Keep the set versioned. Otherwise, a rising aggregate score can conceal a regression in a small but important slice.

For each source, generate center and content-aware rectangles at every production target ratio. Moderate the original and both derivatives. Reviewers should see randomized candidates without the method name, label whether the intended subject survives, and identify any relevant source region omitted from the derivative. This is a paired comparison; using different uploads for each method confounds the result.

A practical decision record should resist a single weighted "quality score":

| Signal | Center crop question | Content-aware question | Release consequence |
|---|---|---|---|
| Subject retention | Is the face or complete dish still legible? | Did the detector choose the intended subject? | Block a method on a harmed slice |
| Moderation coverage | Did cropping remove policy-relevant pixels? | Did saliency steer away from them? | Hold publication on disagreement |
| Stability | Does the same geometry produce the same rectangle? | Does an algorithm change move the rectangle materially? | Require versioned shadow comparison |
| Human correction | How often is the midpoint wrong? | How often is the selected focal region wrong? | Forecast review load |

Moderation coverage comes first because publishing an attractive but incompletely reviewed derivative defeats the job to be done. Subject retention comes next. Stability and correction rate decide whether the operating cost is acceptable. This ordering also prevents a visually pleasing average from compensating for a small set of severe omissions.

Short test sets lie.

They overrepresent clean, well-lit photos because those are easy to curate, while uploads include screenshots, collages, borders, rotated phone images, and compositions in which the intended subject occupies little of the frame. Add cases when reviewers encounter a new failure mode, but preserve a frozen core so repeated evaluations remain comparable.

## Instrument the decision, then control the rollout

Run the proposed selector in shadow mode first. Produce its rectangle and moderation assessment without changing what users see. Capacity planning must include source decoding, candidate generation, moderation of the original plus derivative, durable metadata, and the review queue created by ambiguity. Average latency is insufficient; watch the high percentile that governs the upload SLO and cap concurrent work so a burst cannot exhaust memory with decoded images.

A staged release can then enable the new selector for one content class and one target ratio. Keep the center candidate as a deterministic fallback when content analysis times out or returns an invalid rectangle, but still apply moderation to the source and selected derivative. Roll back by algorithm version, not by silently changing thresholds in place. Reproducibility matters when a reviewer asks why yesterday's crop moved today.

The buy-versus-build decision belongs here because the operational boundary changes what can be measured:

| Option | Coverage evidence | On-call burden | Lock-in surface |
|---|---|---|---|
| Deterministic in-house crop | Rectangle math and logs are fully inspectable | Low compute complexity; quality review remains | Small |
| In-house content-aware pipeline | Detector behavior can be tested by slice | Model serving, rollout, drift checks, and queue capacity stay with the team | Model and runtime choices remain movable |
| Managed image analysis | Require exportable rectangles, versions, and moderation outcomes | Less model hosting; integration and provider incidents remain | Higher if scores or crop semantics are proprietary |

There is no universal winner in that table. A managed capability without reproducible coordinates weakens incident analysis; an in-house detector without staffing for model rollout creates a different reliability problem. Select the boundary that preserves the evidence needed for the coverage SLO and fits the team's actual on-call capacity.

## The threshold has a human cost

A conservative ambiguity threshold sends more avatars and recipe photos to review. Set it too loose and omitted content reaches publication; set it too tight and the queue grows, upload latency rises, and reviewers spend time approving obvious crops. The latter is not harmless. Alert fatigue can reduce attention on the cases that genuinely require judgment.

Treat reviewer capacity as a constrained resource. Measure arrival rate, service time, queue age, and publication timeout separately, then load-test the path with realistic image dimensions and target ratios. If projected review demand exceeds staffed capacity, reduce rollout scope or fall back to the deterministic method for the slices where its measured retention is acceptable. Do not lower the moderation threshold merely to make a dashboard green.

**The release rule is simple: ship content-aware cropping only for slices where it improves subject retention without reducing moderation coverage, and where the resulting review load fits the SLO.** Keep center cropping for predictable compositions and as a failure fallback. The cost of a false positive is not an abstract metric; it is another image in a human queue, another delayed listing, and eventually another alert someone learns to ignore.

## Further reading

- MDN, "Image file type and format guide": https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
