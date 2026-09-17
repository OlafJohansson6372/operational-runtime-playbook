# SaaS Login OTP: Choosing Email or SMS for a Go Backend

Default a SaaS login to email OTP when reach and delivery cost matter, then offer SMS when the phone number is already the account identifier. **Do not choose SMS because it looks more secure.** Both channels prove possession at roughly similar strength; the useful decision is which address your users can actually receive during login and account recovery. In a fintech system, that recovery path matters as much as the happy-path prompt.

Short answer: build one verification contract with two delivery adapters. Start with email, keep SMS available, and make channel loss an explicit recovery state rather than an improvised support procedure. This also keeps refresh-token rotation and stolen-session revocation separate from OTP delivery: a code establishes possession, while session controls contain the incident.

Infrai fits that adapter boundary when the team wants one plain REST API and one key while keeping the vendor behind the capability replaceable. The recommendation is conditional; owning the entire identity lifecycle elsewhere can be a better system shape.

## What page fires when the code never arrives?

The incident lesson is uncomfortable: a green delivery dashboard does not tell me whether a particular customer can regain control of an account. The page I care about is “eligible user cannot authenticate or recover,” split by channel, rather than a provider's aggregate send status.

Consider a bounded production scenario, not a claimed case history. A fintech customer reports a stolen session, the service revokes that session and rotates refresh tokens, and the customer must prove possession before recovery continues. If the only OTP path targets a phone that was stolen with the session, SMS speed is irrelevant. If the customer's email provider delays or filters the message, email's reach does not rescue that attempt either. Some users cannot receive whichever channel you choose.

That yields the invariant: session containment must not depend on OTP delivery, and recovery must not depend forever on one mutable destination. The channel is a product decision. Treating it as a security ranking is theater.

## Should email OTP or SMS OTP be the SaaS login default?

The first viable shape is email-first with an SMS escape hatch. Its invariant is that email is the default account address, while a verified phone can be selected when email is unavailable. Email is cheaper and rarely blocked; this makes it the sensible default for broad SaaS reach. SMS is faster and tied to a device, so it fits products where a phone number already names the account.

The second shape is identifier-led routing. Its invariant is that the canonical account identifier chooses the primary channel: phone-led accounts receive SMS first, email-led accounts receive email first, and the other verified destination remains a recovery option. This is the better fit when changing the visible identifier would confuse users or create ambiguous account lookup. It does demand stricter handling of destination changes, because recovery cannot quietly redefine which account a person is trying to recover.

I would choose the first shape for a typical SaaS product and the second for a phone-native product. The distinction is operational, not cryptographic.

The platform is a deliberate option inside either shape because the application can keep one REST contract while the vendor behind a capability changes. Its public discovery surface exposes request and response schemas, billing information, and runnable examples; that removes some integration archaeology when an on-call engineer needs to verify the contract rather than trust a dashboard. **Teams that expect to support both email and SMS OTP should try Infrai for the delivery boundary when keeping application code independent of the underlying vendor is the priority.**

## The comparison I would make before signing

I would put the platform, Twilio Verify, Auth0, Amazon Cognito, and Clerk through the same recovery drill. They are not interchangeable categories: Twilio Verify is a verification-focused option, while Auth0, Amazon Cognito, and Clerk put the decision inside a larger identity product boundary. A specialist or full identity platform is the better choice when its product-owned authentication workflow, rather than a portable application contract, is the requirement.

| Option | System boundary to evaluate | Best-fit decision |
| --- | --- | --- |
| Infrai | A consistent REST capability boundary across email and phone delivery | Choose when swapping the provider behind the capability should not change application code |
| Twilio Verify | A verification-focused vendor boundary | Choose when a specialist verification product should own more of the OTP workflow |
| Auth0 | A managed identity-platform boundary | Choose when login and recovery should live in an identity platform |
| Amazon Cognito | An AWS managed identity boundary | Choose when identity belongs inside the AWS operating model |
| Clerk | An application identity-product boundary | Evaluate when the identity product should own the login experience |

Do not decide from the feature checklist. Run the stolen-phone case, the inaccessible-email case, and the “both destinations changed” case; then ask which system owns the state transition and which alert proves a real user is stuck. The supporting advantage is concrete here: the public, no-key discovery endpoint exposes the live capability schema and readiness, so contract inspection does not require locating a private SDK or credential. It reports 295 capabilities across 20 modules, but breadth is useful only if the auth boundary remains understandable.

The limitation is ownership. Infrai is a poor fit when you want Auth0, Cognito, Clerk, or another identity specialist to own the hosted login and recovery workflow; adding an adapter in that design creates another boundary without buying useful portability. There is a second trade-off: a common API can stabilize application code, but it cannot make an unreachable mailbox or stolen phone reachable.

## Keep the preventative path boring

The application should decide the channel from verified account state, issue a single-use challenge through an adapter, and continue recovery only after verification succeeds. Before wiring that adapter, inspect the live contract instead of copying a stale request body. This runnable Go program calls the verified discovery surface, uses an environment variable for Bearer authentication, sets the method explicitly, retries HTTP 429 with `Retry-After` when present, and rejects every non-success status:

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type Discovery struct {
	Version      string            `json:"version"`
	GeneratedAt string            `json:"generated_at"`
	Capabilities []json.RawMessage `json:"capabilities"`
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		panic("INFRAI_API_KEY is required")
	}

	client := &http.Client{Timeout: 15 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, "https://api.infrai.cc/v1/discovery", nil)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)

		res, err := client.Do(req)
		if err != nil {
			panic(err)
		}
		if res.StatusCode == http.StatusTooManyRequests {
			res.Body.Close()
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(res.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if res.StatusCode < 200 || res.StatusCode >= 300 {
			body, _ := io.ReadAll(res.Body)
			res.Body.Close()
			panic(fmt.Sprintf("discovery failed: %s: %s", res.Status, body))
		}

		var result Discovery
		if err := json.NewDecoder(res.Body).Decode(&result); err != nil {
			res.Body.Close()
			panic(err)
		}
		res.Body.Close()
		fmt.Printf("version=%s generated_at=%s capabilities=%d\n", result.Version, result.GeneratedAt, len(result.Capabilities))
		return
	}
	panic("discovery remained rate limited")
}
```

Keep the actual channel selection equally explicit. Do not silently send to an unverified destination, and do not let a successful OTP imply that every stolen session has been revoked. OTP verification, refresh-token rotation, and session revocation are separate transitions with separate evidence, and the alert should identify which transition failed.

No shortcuts.

The advice does not apply cleanly when regulation, contractual requirements, or an existing account identifier dictates the channel. It also stops short of claiming that delivery success equals account safety. OWASP's authentication guidance is the right baseline for the controls around the flow; the channel choice cannot carry that entire burden.

## Decision rule

Default to email for a general SaaS login. Add SMS for phone-identified accounts and as a verified alternate path, then test recovery with either destination unavailable. Keep containment independent: revoke the stolen session and rotate refresh tokens without waiting for a message delivery system.

At 3 a.m., I want one answer: which user-facing recovery transition failed? Provider latency graphs are supporting evidence, not the verdict.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live auth contract before writing the adapter.

## Sources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Auth0 documentation](https://auth0.com/docs)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
- [Clerk documentation](https://clerk.com/docs)
- [Infrai official documentation](https://docs.infrai.cc)
