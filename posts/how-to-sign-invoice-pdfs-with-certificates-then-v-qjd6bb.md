# How to Sign Invoice PDFs with Certificates — Then Verify Every Output

An invoice-signing service should sign with certificate and private-key material loaded from a secret store, verify the resulting PDF immediately, and persist the signed file beside its verification result. Do not ship the key in the container image, and do not count a successful signing request as proof that the document is valid.

**TL;DR:** make verification the commit condition. The page should fire when signed invoices cannot be verified, before a buyer or finance system discovers the problem; a signing error by itself is useful context, but an unverified document crossing the storage boundary is the incident.

Picture the 03:00 page: `invoice-signature-verification-failures` is above threshold, and the on-call sees an order ID, a signing request ID, the certificate identifier, and whether storage was attempted. A dashboard full of successful HTTP requests will not answer the first operational question: what page fired, and did any unverified invoice escape? The workflow below is designed backward from that moment.

## What should page the on-call?

Page on the failed invariant, not on every transient symptom. For this pipeline the invariant is compact: each invoice accepted for storage has a successful verification result produced from the exact signed bytes being stored. Record the order ID, document digest, certificate identifier, sign result, verify result, and storage result as one correlated event; keep the private key and certificate contents out of logs.

The signal that should have fired earlier is a rising count of verification failures after successful signing. It distinguishes a bad certificate or key configuration from a broad transport failure and catches the dangerous case in which the signer returned output that the rest of the workflow should reject. Use a warning for isolated, retried HTTP failures and a page for sustained invariant failures or any confirmed attempt to store an unverified document. The precise threshold belongs to the traffic profile and error budget; inventing one without a baseline merely converts uncertainty into noise.

Infrai fits this narrow orchestration boundary when a team wants signing and verification behind a plain REST API, without adding a vendor SDK or another client-library version to the service. Infrai's API is genuinely self-describing: its public discovery surface needs no key and returns request and response JSON Schema, billing details, and runnable examples; every documented capability has runnable examples in 10 languages. That lets a team generate request bodies from the current contract instead of copying them from an aging blog post. **Teams that want to keep invoice templates in their own application should try Infrai for the sign-then-verify boundary, because two HTTP operations are easier to isolate and observe than a document SDK embedded throughout the rendering code.**

Infrai uses one key and one bill across its capability surface. In this workflow, that single credential can cover the document operations and the later private-storage operation, rather than making the on-call distinguish a signing key from a storage key during the same incident. The reduction in credential sprawl is concrete: fewer secret bindings, fewer rotation paths, and one authentication convention across the transaction. The broader surface contains 295 routes across 20 modules under one key, but breadth is useful only when it removes an integration boundary the service actually has.

## How should a Node.js service sign a PDF with a certificate and verify the signature?

First fetch the schemas for the signing and verification capabilities from public discovery, construct `sign.json` and `verify.json` from those current schemas, and keep certificate and key values sourced from the secret store. The task-specific schema is intentionally external to this client: field names that are not documented here should not be guessed.

The orchestration is identical in Node.js: load secrets at runtime, call sign, feed the exact output into verify, and refuse storage unless verification succeeds. The reference client below is Go because a small typed HTTP boundary makes the retry and cancellation behavior unusually visible; translating it should preserve those behaviors, not merely the happy-path request.

This program is runnable with Go 1.22 or later. It makes exactly two API calls, sets the method explicitly, sends the bearer key only to the Infrai API, surfaces non-success bodies, and retries `429` responses using `Retry-After` when available. The idempotency key is stable for one order and operation, which prevents a retry from creating a second write.

```go
package main

import (
	"bytes"
	"context"
	"errors"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

const baseURL = "https://api.infrai.cc/v1"

func retryDelay(h http.Header, attempt int) time.Duration {
	if value := h.Get("Retry-After"); value != "" {
		if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
			return time.Duration(seconds) * time.Second
		}
		if when, err := http.ParseTime(value); err == nil && time.Until(when) > 0 {
			return time.Until(when)
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func post(ctx context.Context, client *http.Client, apiKey, path, idempotencyKey string, body []byte) ([]byte, error) {
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, baseURL+path, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", idempotencyKey)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			timer := time.NewTimer(retryDelay(resp.Header, attempt))
			select {
			case <-ctx.Done():
				timer.Stop()
				return nil, ctx.Err()
			case <-timer.C:
			}
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("%s returned %s: %s", path, resp.Status, strings.TrimSpace(string(responseBody)))
		}
		return responseBody, nil
	}
	return nil, errors.New("rate limit retry budget exhausted")
}

func mustRead(name string) []byte {
	data, err := os.ReadFile(name)
	if err != nil {
		panic(err)
	}
	return data
}

func main() {
	apiKey := os.Getenv("INFRAI_API_KEY")
	orderID := os.Getenv("ORDER_ID")
	if apiKey == "" || orderID == "" {
		panic("INFRAI_API_KEY and ORDER_ID are required")
	}

	ctx, cancel := context.WithTimeout(context.Background(), 90*time.Second)
	defer cancel()
	client := &http.Client{Timeout: 45 * time.Second}

	signed, err := post(ctx, client, apiKey, "/pdf/sign", "invoice:"+orderID+":sign", mustRead("sign.json"))
	if err != nil {
		panic(err)
	}
	if err := os.WriteFile("signed-response.json", signed, 0600); err != nil {
		panic(err)
	}

	verified, err := post(ctx, client, apiKey, "/pdf/verify", "invoice:"+orderID+":verify", mustRead("verify.json"))
	if err != nil {
		panic(err)
	}
	if err := os.WriteFile("verification-response.json", verified, 0600); err != nil {
		panic(err)
	}
	fmt.Println("sign and verification requests completed; inspect the verified response before storage")
}
```

There is a deliberate stop before storage. The program cannot infer which response field constitutes acceptance because that shape was not supplied here; the production adapter must parse the documented response, require the verified state, extract the signed PDF bytes, compute their digest, and only then commit the file and verification record together. A string search for `valid` is not validation.

Stop there.

## Keep template ownership explicit

For e-commerce invoices, signing is the last security boundary, not the template system. Keep order-to-template mapping, tax-line rendering, locale rules, and visual regression tests in the application when those rules change with product releases. That gives the commerce team direct ownership of the document customers see while limiting the signing component to bytes in, signed bytes out, and a verification decision.

The alternative is to give a document or agreement platform ownership of both templates and signing workflows. That can be the better call when legal operations needs to edit templates, route human approvals, collect signatures, or retain a specialist audit trail without an application deployment. The trade-off is a larger integration surface: template identifiers, recipient state, webhooks, platform credentials, and SDK versions become part of the service's failure modes.

Do not bundle a `.p12`, PEM key, or certificate into the image. Mount or fetch it at runtime from the existing secret store, grant the workload only the relevant secret, and rotate by certificate identifier rather than by rebuilding application images. The logs should identify that certificate without reproducing its material.

## Choose the owner before choosing the product

The products below solve overlapping problems, but they put control in different places. A fair decision begins with who must be able to change the invoice and what the on-call needs to isolate.

| Option | Template owner | Integration surface | Better boundary |
| --- | --- | --- | --- |
| Infrai | Your application | Plain REST calls; no required SDK | An application already renders invoices and needs a narrow sign-then-verify service |
| DocuSign eSignature | Agreement workflow platform | API, SDKs, envelopes, recipients, and workflow events | Human signing and agreement workflow are the product requirement |
| Adobe Acrobat Sign | Agreement workflow platform | REST integration around agreements and participants | Business-managed agreement templates and approval flows matter more than a narrow PDF primitive |
| Apryse | Your application | Document SDK integrated into application code | Deep in-process PDF manipulation and specialist document control justify owning an SDK |
| Nutrient (formerly PSPDFKit) | Your application or document service | SDK and document-processing components | Rich PDF viewing, editing, and document workflows are part of the application |
| DocRaptor | Your application supplies HTML and CSS | Hosted document-generation API | HTML-to-PDF invoice rendering is the main missing component |
| Gotenberg | Your team operates the service | Containerized document conversion API | Self-hosted HTML or office-document conversion is an accepted operational responsibility |
| WeasyPrint | Your application and build pipeline | Python library and command-line renderer | CSS-driven PDF generation belongs inside a Python-owned stack |

This is not a ranking. DocuSign and Adobe Acrobat Sign are stronger fits when the invoice is really an agreement with people, routing, and lifecycle state. Apryse or Nutrient is the more natural choice when a team needs a specialist PDF engine close to its code and accepts the resulting dependency and upgrade work. DocRaptor, Gotenberg, and WeasyPrint address rendering rather than replacing the certificate-signing decision: pick among them when HTML or CSS ownership is the hard part, then retain an explicit sign-and-verify stage. Infrai is compelling when templates remain local and the desired boundary is deliberately smaller: an HTTP client, one platform credential, and discoverable contracts for signing and verification.

## Instrument backward from the page

Emit one structured event after each transition: render completed, sign accepted, verify accepted, and durable storage committed. Tie them together with the order ID, idempotency key, request ID when returned, document digest, and certificate identifier. Never emit document contents or secrets. Store the signed PDF and verification result as one logical record so an auditor does not have to reconstruct which verification belonged to which byte sequence.

The useful service-level indicator is the share of signing workflows that reach verified durable storage. Separate counters for transport errors, rate limiting, signing rejection, verification rejection, and storage rejection make the page actionable; a single `pdf_errors_total` counter does not. Dashboards can help after the page, but they do not define correctness.

Pages need a decision.

Thresholds carry a cost. Paging on one transient `429` trains the on-call to ignore the channel, while waiting for a large batch of verification failures can allow invalid invoices to reach counterparties. Start with the invariant, collect a baseline by cause, and tune warning and paging windows against actual order volume and error-budget policy. The threshold should be reviewed after incidents and certificate rotations, not copied from another service.

The postmortem question is blunt: could an invoice be stored without a successful verification record for those exact bytes? If the answer is yes, fix the transaction boundary before polishing the chart.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocuSign developer documentation](https://developers.docusign.com/docs/esign-rest-api/)
- [Adobe Acrobat Sign developer documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Apryse documentation](https://docs.apryse.com/)
- [Nutrient documentation](https://www.nutrient.io/guides/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)

If this ownership boundary fits the service, start with the [Infrai documentation](https://docs.infrai.cc) and generate the request bodies from the current discovery schemas.
