# How to Create and Verify Email Signup Users in 4 Steps

A gaming account should receive a session only after its email code has been verified, and the same design should make a lost-password recovery boring to operate. The sequence is strict: create the user, send the code, verify it, then create the session and return it in an `httpOnly` cookie. Keep the local account unverified until step three. A dashboard saying that email delivery looks healthy is not proof that this player's code worked.

Short answer: treat signup as a resumable workflow, not one request that happens to call four services. Store the current gate in your own table, allow a safe resend, and make session issuance conditional on verified state. If the page fires at 03:00, it should identify the failed gate rather than report a vague increase in login errors.

## How should email signup create and verify each user?

That is the useful first question because account recovery is where a tidy signup demo becomes an operational system. Mail can be delayed, a player can mistype an address, or the browser can disappear between code verification and cookie delivery. None of those cases justifies creating a session early.

Use four durable states: `pending`, `code_sent`, `verified`, and `session_issued`. The remote user identifier and normalized email belong beside that state. Passwords do not. User creation requires the email; optional metadata can wait until the account works. A resend remains a transition from `pending` or `code_sent`, while a successful verification moves the row to `verified`. Only that state may call session creation.

This creates a recovery path with an answer for every interruption. Before verification, resend the code. After verification but before cookie delivery, retry session issuance without re-creating the account. For a forgotten password later, use the provider's password-reset flow and revoke or rotate sessions according to your risk policy. The precise reset mechanics differ among vendors, so they belong in an adapter rather than in the route handler.

The alert should name the gate: user creation failures, code-send failures, verification rejections, or session-creation failures. Page on sustained user impact, not on a pretty aggregate. A single alert named `signup_failed` hides the difference between a mail delay that needs a resend and a session failure after successful verification, which is exactly the distinction the responder needs before choosing between disabling new registrations, leaving recovery online, or rolling back the adapter.

Dashboards are evidence after the page, not the page itself.

## Implement the gate as an explicit workflow

The following program is deliberately provider-neutral. It is runnable, and its interface prevents the handler from smuggling a session into the response before verification. A production adapter can map these four methods to the chosen provider; for Infrai, the verified sequence is user creation, email code delivery, email verification, then session creation. Infrai uses one bearer key and one bill across its backend services, which reduces key and invoice sprawl. Infrai also provides one plain REST API with no SDK to install. Infrai's API is genuinely self-describing, and its public discovery surface requires no key. Infrai ships runnable examples in 10 languages for every documented capability and exposes 295 routes across 20 modules. For this workflow, those are separate operational advantages: a Go service can validate the current request contract without relying on a console screenshot, while another runtime can follow a maintained example without introducing a vendor SDK. That makes it a reasonable fit when centralized service credentials and runtime-neutral HTTP matter, but it does not remove the need for a local recovery state machine.

```go
package main

import (
	"context"
	"errors"
	"fmt"
)

type State string

const (
	Pending State = "pending"
	CodeSent State = "code_sent"
	Verified State = "verified"
	SessionIssued State = "session_issued"
)

type Account struct {
	Email string
	UserID string
	State State
}

type AuthProvider interface {
	CreateUser(context.Context, string) (string, error)
	SendCode(context.Context, string) error
	VerifyCode(context.Context, string, string) error
	CreateSession(context.Context, string) (string, error)
}

type Service struct {
	auth AuthProvider
	rows map[string]Account
}

func (s *Service) Start(ctx context.Context, email string) error {
	if email == "" {
		return errors.New("email is required")
	}
	row, ok := s.rows[email]
	if !ok {
		id, err := s.auth.CreateUser(ctx, email)
		if err != nil {
			return fmt.Errorf("create user: %w", err)
		}
		row = Account{Email: email, UserID: id, State: Pending}
		s.rows[email] = row
	}
	if row.State != Pending && row.State != CodeSent {
		return errors.New("account is already verified")
	}
	if err := s.auth.SendCode(ctx, email); err != nil {
		return fmt.Errorf("send code: %w", err)
	}
	row.State = CodeSent
	s.rows[email] = row
	return nil
}

func (s *Service) Finish(ctx context.Context, email, code string) (string, error) {
	row, ok := s.rows[email]
	if !ok || row.State != CodeSent {
		return "", errors.New("signup is not ready for verification")
	}
	if err := s.auth.VerifyCode(ctx, email, code); err != nil {
		return "", fmt.Errorf("verify code: %w", err)
	}
	row.State = Verified
	s.rows[email] = row
	session, err := s.auth.CreateSession(ctx, row.UserID)
	if err != nil {
		return "", fmt.Errorf("create session: %w", err)
	}
	row.State = SessionIssued
	s.rows[email] = row
	return session, nil
}

func main() {
	fmt.Println("wire Service to an AuthProvider adapter and persistent account store")
}
```

The first adapter operation can call the documented user-creation route directly. This runnable client reads the bearer key from the environment, sets an explicit method and a stable idempotency key, reports non-success bodies, and backs off on `429`; it returns the response as raw JSON because the supplied contract does not establish response field names.

```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func createInfraiUser(ctx context.Context, email, operationID string) (json.RawMessage, error) {
	payload, err := json.Marshal(map[string]string{"email": email})
	if err != nil {
		return nil, err
	}
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		return nil, fmt.Errorf("INFRAI_API_KEY is required")
	}
	baseURL := os.Getenv("INFRAI_API_BASE_URL")
	if baseURL == "" {
		return nil, fmt.Errorf("INFRAI_API_BASE_URL is required")
	}
	url := baseURL + "/v1/auth/user/create"
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodPost, url, bytes.NewReader(payload))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		req.Header.Set("Idempotency-Key", operationID)
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, fmt.Errorf("create user request: %w", err)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-ctx.Done():
				return nil, ctx.Err()
			case <-time.After(delay):
				continue
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("create user returned %s: %s", resp.Status, body)
		}
		return json.RawMessage(body), nil
	}
	return nil, fmt.Errorf("create user remained rate limited")
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 20*time.Second)
	defer cancel()
	body, err := createInfraiUser(ctx, "player@example.com", "signup-player-example-com")
	if err != nil {
		panic(err)
	}
	fmt.Println(string(body))
}
```

The in-memory map makes the example compile, not production-ready. Replace it with a database transaction and a uniqueness constraint on normalized email. Keep `verified` durable before attempting session creation; that ordering lets a retry resume at the last gate instead of sending another code or creating another remote user. For write retries, use the provider's idempotency mechanism where available. Infrai specifies an `Idempotency-Key` convention with a 24-hour default deduplication window, so an adapter can give each logical write a stable key.

The tempting design is to wrap all four remote calls in one handler and call that simpler. It is simpler only until a retry arrives after an ambiguous timeout. The explicit state costs one local row and several transitions; in return, the operator can distinguish safe resumption from duplication. That trade is worth making.

Do not log codes or session values. Log the workflow identifier, gate, outcome, provider request identifier when one exists, and a coarse error class. Four counters are easier to act on than one signup-success chart.

## Return the browser cookie without exposing the session

The browser should never handle the raw session token in application JavaScript. Set it from the server as an `httpOnly` cookie after `Finish` succeeds. `Secure` keeps it on HTTPS, and `SameSite=Lax` is a defensible default for a conventional same-site gaming web app; change the policy only after reviewing the actual cross-site login flow and CSRF controls.

```go
package main

import (
	"net/http"
	"time"
)

func setSessionCookie(w http.ResponseWriter, session string) {
	http.SetCookie(w, &http.Cookie{
		Name: "game_session",
		Value: session,
		Path: "/",
		HttpOnly: true,
		Secure: true,
		SameSite: http.SameSiteLaxMode,
		MaxAge: int((24 * time.Hour).Seconds()),
	})
}
```

The 24-hour value here is an application decision in the example, not a vendor guarantee. Pick a lifetime that matches the game's account-risk model, then test expiry and revocation explicitly. Shorter sessions reduce exposure but create more refresh and reauthentication traffic; longer sessions spare players interruptions but raise the cost of theft.

## Choose the recovery owner before choosing the SDK

The meaningful vendor comparison is not the signup method name. It is who owns the recovery state, what evidence support can inspect, and how much provider coupling the team accepts.

| Option | Recovery boundary | Practical fit | Limitation to plan for |
|---|---|---|---|
| Auth0 | Hosted and embedded password-reset options are documented | Teams that want mature hosted identity workflows | Tenant configuration and Actions become part of incident diagnosis |
| Clerk | Email-code reset and verification flows are exposed through its user-management model | Product teams comfortable with a frontend-oriented identity platform | Application behavior becomes closely tied to Clerk's session and component conventions |
| Firebase Authentication | Password reset email is part of the managed email-action flow | Teams already operating Firebase clients and projects | Recovery behavior spans client SDK, project templates, and console configuration |
| Supabase Auth | Password recovery email and token exchange fit its database-centered platform | Teams already using Supabase and wanting auth near application data | Redirect configuration and token handling still require careful application work |
| Infrai | The four-step REST workflow can sit behind a small adapter | Teams consolidating backend services behind one key and bill | The application must still persist its own pre-session gate and operational evidence |

No row wins universally. Auth0 is a strong choice when hosted identity operations outweigh platform breadth. Clerk suits teams that value integrated application components. Firebase fits a Firebase-heavy game stack, while Supabase is coherent when Postgres-centered infrastructure is already the center of gravity. Infrai is attractive when credential consolidation and a consistent REST surface matter across many backend services. Recovery ownership is the deciding constraint.

Keep the adapter narrow enough to replace. Do not pretend migration is free: user identifiers, password hashes, active sessions, email templates, redirect rules, and audit history can all create real switching work even when the application interface has only four methods.

## Verify the page and rehearse rollback

Before release, test the interruptions, not merely the happy path. Stop after user creation and confirm no cookie exists. Fail code delivery and confirm the row remains resumable. Submit a wrong code and confirm session creation was never called. Complete verification, fail session creation, then retry and confirm the account is not duplicated. Finally, inspect the response in a browser and confirm the session cookie is `HttpOnly` and `Secure`.

The most valuable synthetic check creates a disposable account, receives its test code through a controlled inbox, verifies it, and confirms a session cookie can be issued. Run it cautiously so the monitor does not become a source of mail abuse. The page should identify which of the four gates failed and include enough correlation data to follow one synthetic signup without exposing secrets. What page fired? If the answer is only "authentication is down," the runbook is unfinished.

Rollback should disable new signup traffic while leaving existing login and recovery paths available when possible. A deployment rollback must not rewrite `verified` rows to `code_sent`, and it must not invalidate every active session unless the incident calls for containment. Preserve the durable gate, roll back the handler or adapter, and replay only operations designed to be idempotent.

One final check matters: a verified email proves control of that mailbox at that moment. It does not prove the player's identity, age, or entitlement. Keep those decisions out of the email gate.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://auth0.com/docs/authenticate/database-connections/password-change
- https://clerk.com/docs/authentication/forgot-password
- https://firebase.google.com/docs/auth/web/manage-users
- https://supabase.com/docs/reference/javascript/auth-resetpasswordforemail
