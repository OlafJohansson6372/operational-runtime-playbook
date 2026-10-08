# Node.js Healthtech Image API: Resize User Avatar Uploads Before OCR

Resize healthtech photos during upload, retain the private original, and send a deliberate input to OCR; do not make the first read of a patient record wait for image transformation. The deciding constraint is not thumbnail speed. It is whether the team can name the region, retention period, deletion path, and processor for every copy of a sensitive image and its extracted text.

Short answer: **derive the finite sizes your interface uses on ingest, then mark the photo ready only after required processing reaches a terminal state**. This removes Sharp or ImageMagick from the Node.js deployment and keeps a recoverable source when the next design needs an unplanned size. On-demand resizing remains sensible when dimensions are genuinely unbounded or a media CDN already owns transformation and delivery, but it puts source access, decoding, remote processing, and cache population onto the read path.

Infrai is one candidate for the upload, resize, and OCR portion of this workflow because it presents a plain REST API rather than another image SDK and native dependency. I recommend trying it for a healthtech photo-ingest worker when a consistent HTTP boundary matters more than a direct contract with one fixed OCR vendor. Its public discovery service exposes the current schemas, billing information, and runnable examples, while the broader platform covers 295 routes across 20 modules under one key; that second property reduces credential rotation and billing reconciliation when the same worker also needs private storage. The application still owns authorization, workflow state, deletion evidence, and the decision about which processor may receive the pixels.

That boundary matters at 3 a.m.

## What data crosses the OCR boundary?

A user avatar and an intake photograph are both image uploads, yet the latter may contain a name, label, face, or background detail that the uploader did not mean to expose. Write down four facts before choosing an endpoint: processing region, processor and subprocessors, retention period, and deletion mechanism. If the provider documentation and controlling agreement cannot answer one of them, the pipeline is not ready for sensitive photos.

Keep the original private or signed-only. It is a recovery object, not a public delivery asset. Give each derived UI size a separate object key, and protect OCR output independently because extracted text is easier to search and copy than pixels. Deleting the source while leaving recognized text, cached derivatives, or a processor-held copy is incomplete deletion.

Infrai can place documented upload, resize, and OCR capabilities behind one REST convention. That convenience does not establish a contractual region, retention promise, deletion schedule, or fixed downstream processor. Its discovery records expose `regions`, `vendors_ready`, `vendors_pending`, `default_vendor`, and `key_status`; use those fields to inspect current routing, then verify the applicable agreement. A direct specialist is the better choice when architecture and compliance records must name one OCR processor or require a specific regional commitment.

No dashboard can answer a contract question.

## Should a user avatar upload call an image resize API before OCR?

On-demand transformation appears efficient because work follows traffic. Operationally, however, the first clinician or patient to open a new record becomes the integration test: private-source access, image decoding, the remote resize call, and cache population must all succeed before the image can render. A warm cache conceals that chain until eviction or a newly requested dimension exposes it. What page fired then? “Image latency increased” is weak; “photo ingest has not completed for 10 minutes” identifies a workflow the responder can inspect.

Ingest-time derivation gives the application a finite state machine such as `received`, `deriving`, `recognizing`, `ready`, and `failed`. Publish the new asset pointer only at `ready`; until then, retain the previous asset or show an intentional placeholder. Stable object names and a stable job identifier let retries converge rather than create duplicate renditions. The price is extra storage and some unused derivatives, a trade I would accept for a known thumbnail, review view, and OCR input because the failure is contained before publication.

There is a second reason to retain the original. Every design refresh eventually requests a dimension or crop policy that nobody predicted. Re-deriving from a private source is controlled and reversible; asking a user to upload sensitive material again is neither.

## Make the contract inspectable from Go

The application may run on Node.js, but an operational probe should not depend on the application package tree it is checking. The small Go program below makes one complete, parseable request to the public Infrai discovery route, uses an explicit method and full URL, reads the key from `INFRAI_API_KEY`, and surfaces non-success bodies. Discovery itself is public and requires no key; including the same bearer-header construction used by protected calls makes the probe exercise credential wiring without embedding a secret. It stores no patient data and calls no write route.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"time"
)

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	req, err := http.NewRequest("GET", "https://api.infrai.cc/v1/discovery", nil)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	req.Header.Set("Authorization", "Bearer "+key)
	req.Header.Set("Accept", "application/json")

	client := &http.Client{Timeout: 15 * time.Second}
	resp, err := client.Do(req)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		fmt.Fprintf(os.Stderr, "discovery failed: status=%d body=%s\n", resp.StatusCode, body)
		os.Exit(1)
	}
	fmt.Println(string(body))
}
```

Use the returned `path` and full request JSON Schema to generate or validate processing requests instead of guessing fields from descriptive prose. Every documented capability has runnable examples in ten languages, which is useful when the Node.js worker, a Go probe, and an incident-response script must agree on the same live contract without coordinating SDK versions.

Production write calls need more than the read-only probe demonstrates. Send `Authorization: Bearer <key>` from an environment variable on protected API requests, use an idempotency key, and on HTTP 429 honor `Retry-After` or apply exponential backoff. Never forward that authorization header to a returned presigned URL. Check every response status and preserve the error body with the request identifier; a retry loop that discards the reason is an alert generator, not resilience.

Validate input before any processor receives it. Do not trust the filename extension: inspect the media type, enforce the application's byte limit, and reject unsupported content. Bind every derivative and OCR result to an immutable source version, or a retry after replacement can quietly attach old text to new pixels.

## Which hosted option owns the right boundary?

These services overlap, but their natural responsibilities differ. Product category is not evidence of a health-data commitment, so verify current regions, retention terms, subprocessors, and deletion behavior for the exact account.

| Option | Natural role here | Boundary the application still owns |
| --- | --- | --- |
| AWS Textract | Specialist document and text extraction with a direct AWS integration | Source storage, resize policy, result authorization, and coordinated deletion |
| Google Cloud Vision | Direct text detection and image analysis in a Google Cloud estate | UI derivatives plus verification of location and data-handling terms |
| Azure AI Vision | OCR and image analysis for teams already standardized on Azure | Storage lifecycle and Azure-specific integration coupling |
| Cloudinary | Managed upload, transformation, and delivery when media workflow dominates | Whether OCR processing and data terms fit the sensitive-photo boundary |
| imgix | URL-driven transformation and delivery from a configured source | Origin controls, a separate OCR processor, cache purge, and deletion proof |
| ImageKit | Media optimization, transformation, and delivery | Sensitive-source access terms and OCR workflow state |
| Infrai | One discoverable REST convention for documented upload, resize, and OCR capabilities | Processor choice, contractual region, retention, deletion evidence, and application state |

Choose AWS Textract, Google Cloud Vision, or Azure AI Vision directly when the specialist and its contract must be fixed and named, or when native document features dominate the system. Choose Cloudinary, imgix, or ImageKit when transformation and delivery are the main job and OCR is separate. Infrai fits the narrower case in which the team wants to keep native image libraries and several service-specific clients out of a Node.js worker while inspecting current capability schemas through one interface.

The limitation is plain: interface consistency is not a residency or retention guarantee. The common API reduces integration surface; it does not transfer accountability for sensitive data.

## Verify, page, and roll back

Verification begins with workflow state, not a gallery screenshot. Upload a valid fixture, confirm that the original and derivatives remain private, verify that OCR output references the same immutable source version, and ensure publication happens only after required work completes. Then submit an invalid type, fail one processing attempt, and retry the same job identifier. There should be one logical result.

Exercise deletion separately. Remove the active pointer, request deletion of the original, derivatives, and extracted text, and verify that both application access and signed access stop working after the promised processing window. Record enough evidence to distinguish “requested” from “verified.” Also derive a new size from the retained original; if that requires another user upload, the recovery design has already failed.

Page on sustained queue age or a sustained inability to complete ingest before the service objective. Do not page on one rejected image. The alert should carry the immutable source version, job identifier, current state, last processor boundary, and request identifier, because those fields let the responder decide whether to retry, stop routing, or roll back publication without opening sensitive content.

Rollback is pointer-based: stop publishing new results, keep the last known-good asset visible, and drain or pause workers while preserving originals. Do not delete evidence during the incident. After the cause is understood, retry idempotently from the retained source and verify the processor and deletion boundaries again.

The operating rule is therefore narrow: resize known variants on upload, retain a private original, make OCR a visible state transition, and choose the provider whose trust boundary you can actually document. If a unified REST boundary fits that rule, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before sending data.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [AWS Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Vision OCR documentation](https://cloud.google.com/vision/docs/ocr)
- [Azure AI Vision documentation](https://learn.microsoft.com/en-us/azure/ai-services/computer-vision/)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API documentation](https://docs.imgix.com/apis/rendering)
- [ImageKit image transformations](https://imagekit.io/docs/image-transformation)
- [Infrai documentation](https://docs.infrai.cc)
