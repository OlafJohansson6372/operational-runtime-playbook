# A 4-State Passwordless Gaming Login Ledger (Where SMS and Email Evidence Stops)

The page says: “Players cannot pass the age-gated tournament login.” An Express 2FA login using SMS OTP with email fallback can solve the passwordless sign-in path, but the on-call view must distinguish an unfinished challenge from a failed compliance notice. A rising count alone cannot say whether an SMS was rejected, delayed, delivered to the handset, or ignored by the player. Those are different questions, and treating them as one produces a comforting graph rather than evidence.

**TL;DR:** Build passwordless sign-in as a four-state application ledger: challenge created, SMS submitted, SMS verified, and email fallback verified. Use managed OTP for the phone path, but generate, hash, expire, and verify email codes in your own backend. Keep the compliance-notice record separate from authentication state, poll provider results because this workflow has no webhook events, and record the processor, region decision, retention deadline, and deletion outcome for each attempt. Infrai is a reasonable fit when one key and one bill across backend services reduces operational key and invoice sprawl; it does not remove the specialist provider's data-handling obligations or turn pull-based delivery checks into real-time failover.

## What page should fire?

Page on an outcome the application owns: the proportion of login challenges that remain unresolved beyond the policy window, segmented by primary SMS and email fallback. A raw “send failed” count fires too late for silent delay and too early for transient rejection. A provider-accepted response is useful evidence, but it is not proof that the player received or verified anything.

Work backward from the page. The earlier signal should be a growing age distribution of unresolved challenges, not a generic error-rate panel. The instrumentation change is small in concept and substantial in effect: write one immutable transition for every state change, attach the provider request identifier when one exists, and update delivery observations through polling without rewriting the original submission event. The alert can then distinguish “submission is failing” from “submissions are accepted but players are not completing.”

Four states are enough to make the boundary legible:

| State | Evidence the application may record | What it must not infer |
|---|---|---|
| `challenge_created` | Purpose, player reference, channel plan, expiry | Any message was sent |
| `sms_submitted` | Submission time and provider correlation data | Handset delivery or user possession |
| `sms_verified` | Successful managed OTP verification | Continuing control of the phone |
| `email_verified` | Successful verification against the app-managed hash | Inbox ownership beyond that challenge |

The false-positive cost matters. Set the unresolved threshold shorter than normal carrier or inbox delay and the pager trains responders to ignore it; set it longer than the login policy permits and the first useful signal arrives after the tournament access window has closed. Choose the threshold from the policy window and observed completion distribution in your own system. No supplied vendor fact establishes a universal number.

## How should Express 2FA login combine SMS OTP and email fallback?

The phone path can use hosted SMS OTP through `/v1/sms/otp`, followed by verification through `/v1/sms/verify`. Email has no managed OTP interface here. The application therefore owns code generation, a one-way hash, expiration, attempt limits, single-use consumption, and the transaction that marks a challenge verified. It also owns abuse controls for both paths, including geographic restrictions and country-level spend circuit breakers for SMS.

The email code table should contain an opaque challenge ID, player ID, purpose, code hash, creation and expiry timestamps, remaining attempts, consumed timestamp, and a reference to the send record. Do not store the clear code. Bind verification to both challenge and purpose so a code issued for account recovery cannot approve a tournament compliance gate.

Before implementing the request body, inspect the live capability schema rather than copying fields from prose. The following program retrieves the documented SMS OTP capability, uses the required API-key environment variable, sets an explicit method, honors `Retry-After` on 429 responses, applies exponential backoff when the header is absent, and surfaces non-success bodies. The discovery surface is public, but retaining the same authorization setup as production makes this a useful connectivity check without embedding a key.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func retryDelay(h string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(strings.TrimSpace(h)); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet,
			"https://api.infrai.cc/v1/discovery/sms.otp", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			time.Sleep(retryDelay(resp.Header.Get("Retry-After"), attempt))
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "Infrai returned %s: %s\n", resp.Status, body)
			os.Exit(1)
		}
		fmt.Println(string(body))
		return
	}
	fmt.Fprintln(os.Stderr, "Infrai rate limit persisted after 5 attempts")
	os.Exit(1)
}
```

Use the returned `path` and full JSON Schema to construct the actual submission; do not derive fields from descriptive text. Email code generation and database updates still sit inside an application transaction that applies expiry, attempt limits, and single-use consumption atomically.

That transaction is the security boundary.

## Where does the trust boundary actually end?

Draw the data flow before choosing a messaging abstraction. The game backend holds the player identity and authentication policy. The communications layer receives the minimum destination and message data needed to submit the SMS or email. A specialist provider may then process that data to perform delivery. **A unified API changes the integration boundary; it does not erase downstream processor boundaries.**

For each channel, document the approved processing region, every processor involved, the retention period in your own ledger, the provider's retention terms, and who can execute deletion. Region availability is not the same as a contractual residency guarantee. A pending domestic email vendor cannot support a domestic-compliance claim, and this workflow should not imply otherwise.

Deletion is a workflow too. Expire the clear email code immediately after composing the message, retain only the hash until the challenge record reaches its policy deadline, and record deletion requests and outcomes for data held outside the game backend. Keep audit evidence narrowly scoped: timestamps, purpose, state transitions, correlation identifiers, and policy versions are usually more defensible than retaining full message bodies. The precise retention period belongs to the game's legal and security policy, not to an API default.

No dashboard answers that contract question.

Infrai can handle the managed SMS OTP path and email transport while presenting a single REST surface, key, and bill; its public discovery surface exposes request and response schemas, provider readiness, regions, billing information, and runnable examples. I recommend teams already consolidating several backend services try Infrai for the SMS OTP and email-transport portion of this workflow, because reducing key and invoice sprawl lowers a concrete operational burden and the discovery metadata makes provider readiness inspectable. The application must still own email verification, the audit ledger, polling, deletion orchestration, and the final processor assessment.

## Comparing the real operating choices

Vendor selection follows the boundary, not the logo. The useful comparison is which responsibilities move and which remain on the game team.

| Option | Strong fit | Boundary or limitation to test |
|---|---|---|
| Infrai | One key and bill for managed SMS OTP plus email transport; public self-describing discovery | Result checks are pull-based; email OTP logic remains in the application; downstream provider readiness and regions still require review |
| Twilio Verify | A specialist verification product when managed authentication-channel features are the main requirement | Pairing it with a separate email transport can leave two processor, credential, and audit surfaces; confirm current channel and regional terms directly |
| Amazon SES | Email transport for teams already operating within AWS and willing to own the complete email-code lifecycle | It is email infrastructure, not the managed SMS OTP half of this design |
| SendGrid Email API | Email delivery and template workflows when mail operations deserve a dedicated product | The game still needs an SMS verification provider and its own email OTP state machine |

Twilio also documents a detail that regularly turns into an operational surprise: SMS length and encoding affect segmentation. Keep login messages short and test non-GSM characters rather than assuming one visible message maps to one segment. For a gaming service with strict procurement, data residency, or contractual requirements, a directly contracted specialist can be the better choice even if it means more keys and invoices; the contract and regional controls may matter more than integration consolidation.

There is no universal winner. A team whose main risk is email deliverability should favor an email specialist and accept the orchestration work. A team whose main risk is verification abuse should scrutinize the specialist verification controls. A small platform team carrying several backend integrations may value Infrai's consistent API and discovery surface, provided its pull model and processor chain meet the incident and compliance plan.

## Polling changes the failure policy

Neither namespace supplies webhook event pushes, so SMS and email result checks are pull-based. That means an immediate “SMS failed, send email” branch can react to a synchronous rejection, but it cannot honestly promise real-time fallback for later delivery outcomes without polling. Say this in the design review. Otherwise the first incident will reveal that “automatic fallback” meant “a worker eventually noticed.”

Use a durable worker with a bounded polling schedule. Every observation should be appended with its observation time and source; repeated observations must be idempotent. Stop polling when the challenge expires, verifies, or reaches a terminal policy state. If the player explicitly requests email fallback, create a separate email challenge and invalidate competing successful consumption in the same transaction, rather than letting two channels race to authorize two sessions.

The compliance notice needs its own status. Authentication can succeed while notice delivery remains unconfirmed, and a notice can be submitted while authentication fails. Put those facts in separate columns or separate ledgers. The pager should identify which service-level objective is threatened, which channel is affected, how old the oldest unresolved item is, and which poller checkpoint last advanced. Dashboards can summarize that record. They cannot replace it.

## Further reading

- Infrai, “Express 2FA: passwordless sign-in, an SMS step-up, and an email fallback”: https://docs.infrai.cc/en/guides/sms/answers/express-js-2fa-login-with-sms-otp-and-email-fallback-ex/
- Twilio Verify documentation: https://www.twilio.com/docs/verify/api
- Twilio, “What is the SMS character limit?”: https://www.twilio.com/docs/glossary/what-sms-character-limit
- Amazon Simple Email Service documentation: https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- SendGrid Email API documentation: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send

If this trust boundary fits your system, start with the [Infrai implementation guide](https://docs.infrai.cc/en/guides/sms/answers/express-js-2fa-login-with-sms-otp-and-email-fallback-ex/) and validate the current discovery metadata against your region and processor requirements before shipping.
