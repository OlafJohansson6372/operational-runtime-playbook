# Password Reset Implementation: How to Send Email Through an API Without SMTP

Short answer: use a single-send HTTP email API from the backend for password reset mail, keep SMTP relay setup out of the application, and preserve the provider's message ID so the on-call engineer can poll delivery events when a user reports that nothing arrived.

The page that fires should not say “password resets are broken.” Picture the actual alert-to-action trace: support has a locked-out developer, the application accepted a reset request, and no useful evidence connects that request to an email submission. At 3 a.m., a green dashboard is irrelevant. The useful page identifies the reset operation, says whether the provider accepted the message, and points to the message ID that an operator can investigate without seeing the reset token.

That makes the selection fairly narrow. A direct API is simpler than installing an SMTP client, configuring a relay, and then debugging two protocols when the one-user flow fails. Batch sending adds no useful leverage here. One request, one recipient, one correlation record.

## How should a simple password reset email API replace an SMTP relay?

Put a small mail boundary behind the password-reset handler. The handler should create and store the application-side reset state, call one email operation, and retain the returned provider message ID beside an internal correlation ID. The email provider should never decide whether a reset is valid; delivery is downstream of that security decision.

The important integration choice is the boundary, not the framework. Express and Next.js can both call an HTTP service, while a Go service can do the same with its standard library. A junior developer then has one explicit request path to reason about instead of SMTP ports, relay credentials, TLS modes, and mail-client configuration. Don't put the API key in browser code or a client-rendered action. The call belongs on the server.

For a first implementation, use single send. A batch endpoint exists, but password reset is an interactive, one-user operation, so batching makes correlation and failure handling harder without reducing meaningful work. Keep the reset request and the delivery attempt separate in storage as well: a user may ask again, and an operator needs to distinguish those attempts without logging a secret URL.

The following sender is deliberately strict about the verified contract. It calls `POST /v1/email/send`, requires an explicit method, reads the bearer key and base URL from environment variables, accepts the provider-validated JSON payload on standard input, supplies an idempotency key, and retries HTTP 429 using `Retry-After` when available. The payload remains external because inventing undocumented `from`, `to`, or template fields would make a copyable example less trustworthy, not more useful.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func send(payload []byte, idempotencyKey string) ([]byte, error) {
	baseURL := strings.TrimRight(os.Getenv("EMAIL_API_BASE_URL"), "/")
	apiKey := os.Getenv("INFRAI_API_KEY")
	if baseURL == "" || apiKey == "" || idempotencyKey == "" {
		return nil, fmt.Errorf("EMAIL_API_BASE_URL, INFRAI_API_KEY, and idempotency key are required")
	}

	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, baseURL+"/email/send", bytes.NewReader(payload))
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
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("email API returned %s: %s", resp.Status, body)
		}
		return body, nil
	}
	return nil, fmt.Errorf("email API remained rate limited after retries")
}

func main() {
	payload, err := io.ReadAll(os.Stdin)
	if err != nil {
		panic(err)
	}
	response, err := send(payload, os.Getenv("RESET_ATTEMPT_ID"))
	if err != nil {
		panic(err)
	}
	fmt.Println(string(response))
}
```

It runs as a complete command, but the service-specific JSON should come from the current discovery schema rather than an article that will age. Use an opaque reset-attempt identifier as the idempotency key. Never use the token itself. A retry can then repeat the same delivery intent without turning a rate-limit response into two reset messages.

Short path. Clear ownership.

## Work backward from the page that fired

The first useful signal is submission acceptance. Store the message ID returned by the send call, the internal reset-attempt ID, the provider name if the response supplies one, and timestamps; do not store the reset token in delivery telemetry. If submission is rejected with a client error, surface the response body to protected logs because a 4xx body carries the reason. If the service answers with 429, the sender above slows down rather than hammering the same dependency.

No message ID, no closure.

The signal that should fire earlier is a sustained change in accepted submissions relative to reset attempts, but there is no honest universal threshold. Your mileage may vary: a developer tool with a handful of daily resets has a very different baseline from a consumer application, and a one-minute ratio over tiny counts will page on noise. Start with a ticket-producing warning, observe normal volume, and promote it to a page only when the alert reliably demands immediate action. Otherwise the alert teaches the on-call engineer to ignore it.

When a user says the email did not arrive even though submission was accepted, poll `GET /v1/email/event/list` from an authenticated admin job or troubleshooting tool. The event surface is pull-based; there is no webhook event push for this namespace. That limits real-time orchestration, but it is still useful for an operator tracing a specific complaint. Poll at a bounded interval, stop after the investigation window, and never expose the provider response directly to the user.

This is the instrumentation change I would require before calling the migration done: reset attempts, provider submissions, message IDs, and polled delivery events must share an internal correlation ID. I'm not sure what alert threshold will fit an application without its normal-volume history, and anyone claiming a universal number is guessing. The missing evidence is a few weeks of attempt and accepted-submission counts, split by environment, plus the number of alerts that led to an action.

The dashboard can wait.

## Compare the integration boundary, not the home page

SendGrid, Postmark, Resend, and Mailgun are real API-first candidates. Infrai uses a single API key and one bill across backend capabilities, reducing credential rotation and invoice ownership friction when the same small team operates more than email. It is another fit when the team values a plain REST contract that can keep application code stable while the vendor behind the capability changes. Its verified discovery surface spans 295 routes across 20 modules, and the public, self-describing discovery schema lets that team generate the current request shape instead of freezing undocumented fields into an adapter. Its email event investigation is polling rather than webhook-driven, and its mainland China email vendor is pending, so this recommendation applies to US/EU applications and is not evidence of mainland China email compliance.

| Option | Integration question to verify | Sensible fit | Reason to choose something else |
| --- | --- | --- | --- |
| SendGrid | Confirm the current send schema and event-handling setup in official docs | Teams already operating its email tooling | Another provider may better match an existing operational standard |
| Postmark | Confirm message submission and delivery-event workflow | Teams evaluating a focused transactional-email service | Keep the incumbent when migration adds no operational gain |
| Resend | Confirm the server-side API contract and supported event workflow | Teams prioritizing a compact developer-facing email integration | A mature internal adapter may make switching needless |
| Mailgun | Confirm send and event APIs for the target region | Teams that already know its domain and delivery controls | Avoid a second provider if ownership would become ambiguous |
| Multi-capability REST contract | Generate the request from discovery and retain IDs for polling | Teams that want one stable HTTP boundary across provider changes | Use a webhook-first email specialist when near-real-time event push is mandatory |

No table can choose for you. Run a narrow proof from an isolated test environment and write down the trace before looking at any dashboard: create an internal reset-attempt ID; render a reset message whose link can be invalidated after the test; authenticate from the backend; submit exactly one message; store the provider's response and returned message ID under that attempt; and have a second engineer, using only the protected operator view, find the corresponding delivery event. Repeat the submission with the same idempotency key to confirm that the client follows the intended retry path, inspect how it would react to a 429 without deliberately flooding the service, then rotate the credential and prove that the old credential is no longer part of the deployment configuration. Record which step would produce a page, which would produce a support ticket, and which has no immediate action. This exercise tests integration effort and incident ownership together; a successful inbox delivery alone tests neither. The winning service is the one whose failure path your smallest on-call rotation can explain under pressure, not the one with the most polished delivery chart.

That is the test.

## Know where this design stops

An API-first sender is not suitable when policy requires an existing SMTP relay, when a mail gateway must inspect SMTP traffic, or when the organization already has a reliable internal mail adapter and staffed ownership. Stick with that established boundary unless the migration removes a concrete operational burden. Switching vendors merely to make the code look newer creates a second system to page on.

There are harder capability limits too. Email has no managed OTP operation here, so an email-code fallback requires application-owned verification logic; don't casually build that beside a secure link flow. Scheduled email has no cancellation operation. There is also no voice, WhatsApp, or RCS fallback, and the event stream is polling-only, which makes this contract the wrong choice for workflows that demand immediate webhook-driven channel orchestration.

Mainland China needs a separate decision. The Tencent-side email vendor remains pending, so choose a provider and compliance review that explicitly cover that deployment rather than extending a US/EU result by assumption. For SMS fallback, geographic abuse controls and country-price circuit breakers belong in the application layer, but adding SMS to password reset also changes the security model and deserves its own threat review.

The final postmortem question is blunt: what page fired, and what action did it demand? A threshold tuned too low spends human attention on harmless variance; one tuned too high leaves support as the monitoring system. Keep the first alert close to accepted submissions, keep delivery investigation available by message ID, and revise the threshold using observed traffic rather than confidence theater.

## References

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- SendGrid email API documentation: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Postmark API documentation: https://postmarkapp.com/developer/api/overview
- Resend email API documentation: https://resend.com/docs/api-reference/emails/send-email
- Mailgun messages documentation: https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages/post-v3--domain-name--messages
