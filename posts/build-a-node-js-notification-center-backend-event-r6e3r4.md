# Build a Node.js Notification Center Backend — Event Audit and Polling History

Short answer: build the Node.js notification center backend around an event outbox, attempt-level audit log, and polling API for email and SMS delivery history; report what was accepted, delivered, suppressed, or still unknown. The integration decision should follow the evidence your B2B SaaS customers must inspect, not the sender API that looks easiest in a demo.

## The incident signal is a missing receipt, not a failed request

At 3am, a green dashboard saying “email accepted” is weak evidence. Accepted means the downstream service took a request; it does not prove a mailbox or carrier delivered it. A notification center needs two durable facts: the immutable notice (why it was sent and to whom) and each delivery attempt (which channel, provider reference, timestamps, and terminal state).

For a B2B SaaS account-change notice, create one event keyed by `account_id` and a policy version. Render channel-specific content from that event, but never let a retry mutate the original text. Store recipient addresses encrypted or tokenized, retain a hash for matching, and redact message bodies from ordinary operational logs. A compliance reviewer should be able to answer “what page fired?” without receiving a dump of personal data.

The state machine can stay small: `queued`, `accepted`, `delivered`, `failed`, `suppressed`, and `unknown`. `unknown` matters after a worker dies between a network write and its database commit. Calling that case `failed` creates a false record; calling it `delivered` creates a worse one.

## How should a Node.js notification center model email SMS audit log and delivery history?

Use an append-only attempt table and an outbox lease. The dispatcher claims an event, sends one channel, and writes the provider reference in the same idempotency scope. A separate reconciler polls status until a terminal state or a policy deadline, while the API serves a stable cursor over attempts. Webhooks can shorten the interval, but polling remains the recovery path when an event is delayed or dropped.

Here is a deliberately boring worker sketch. The transport is an interface so the policy is testable without a live sender.

```go
package notify

import (
	"context"
	"errors"
	"time"
)

type State string

const (
	Queued State = "queued"
	Accepted State = "accepted"
	Delivered State = "delivered"
	Failed State = "failed"
	Suppressed State = "suppressed"
	Unknown State = "unknown"
)

type Attempt struct {
	EventID string
	Channel string
	Key string
	State State
	RemoteID string
	UpdatedAt time.Time
}

type Sender interface {
	Send(context.Context, string, string, string) (remoteID string, state State, err error)
	Status(context.Context, string) (State, error)
}

type Store interface {
	Claim(context.Context, string) (Attempt, bool, error)
	Save(context.Context, Attempt) error
}

func Dispatch(ctx context.Context, store Store, sender Sender, eventID, channel, body string) error {
	attempt, claimed, err := store.Claim(ctx, eventID)
	if err != nil || !claimed { return err }
	attempt.Channel, attempt.Key = channel, eventID+":"+channel
	remote, state, err := sender.Send(ctx, attempt.Key, channel, body)
	if err != nil {
		attempt.State, attempt.UpdatedAt = Unknown, time.Now().UTC()
		_ = store.Save(ctx, attempt)
		return err
	}
	attempt.RemoteID, attempt.State, attempt.UpdatedAt = remote, state, time.Now().UTC()
	return store.Save(ctx, attempt)
}

func Reconcile(ctx context.Context, store Store, sender Sender, attempt Attempt, deadline time.Time) error {
	if attempt.RemoteID == "" || attempt.State == Delivered || attempt.State == Failed || attempt.State == Suppressed { return nil }
	for time.Now().Before(deadline) {
		state, err := sender.Status(ctx, attempt.RemoteID)
		if err != nil { return err }
		attempt.State, attempt.UpdatedAt = state, time.Now().UTC()
		if err := store.Save(ctx, attempt); err != nil { return err }
		if state != Queued && state != Accepted { return nil }
		time.Sleep(2 * time.Second)
	}
	attempt.State = Unknown
	return store.Save(ctx, attempt)
}

var ErrNoAttempt = errors.New("no attempt")
```

The exact sender implementation is replaceable; the record contract is not. Keep idempotency keys deterministic (`event_id:channel`) and enforce uniqueness in the database. On restart, a worker can safely re-read an `unknown` attempt and reconcile it rather than issuing an unbounded resend.

## Polling API details that survive retries and deploys

Expose `GET /notifications/{id}` for a single notice and `GET /notifications?cursor=...&limit=...` for history. Return the server's opaque cursor, not a timestamp assembled by the client. Sort by `(created_at, attempt_id)` so two records with the same clock value do not reorder. Include `next_cursor`, `state`, `channel`, `created_at`, `updated_at`, and a redacted reason; omit message bodies by default.

Clients should poll with backoff and stop on a terminal state. A 202-style “accepted” response is useful for the create operation, but it must not be copied into the delivery history as proof. Add a request ID to every response and log it with the event ID; that pair is what an incident responder can search when the page says a notice is missing.

One trap took me a while to name: a cursor that encodes only `created_at` skipped an attempt during a same-millisecond batch. The fix was a compound cursor and a uniqueness test, not a larger polling interval. I found it while replaying a deployment where three account notices entered the outbox in one database transaction; the first worker acknowledged two remote requests, lost its lease, and a replacement worker resumed from the timestamp cursor. The API showed two attempts, the audit export showed three, and support understandably treated the mismatch as a missing notice. We added the attempt ID to the cursor, made ordering deterministic, and replayed the fixture across clock skew and worker restarts before shipping. That test now runs on every schema change because an audit trail that changes shape under load is not an audit trail.

Page fired.

## Verification, rollback, and the limits of this design

Test the state transitions with a fake clock: accepted then delivered, accepted then failed, suppression before send, and a worker crash after the remote response. Assert that replaying an event creates at most one attempt per channel. Run a contract test against each sender's sandbox and compare its status vocabulary to your internal enum; map unknown values to `unknown`, never to success.

Deploy the reader first, then the writer, and keep a feature flag that pauses new sends while allowing reconciliation and history reads. Roll back code without deleting attempts. If a schema change is unavoidable, add nullable fields and backfill asynchronously; audit evidence should remain readable during the rollback window.

The catch is that this architecture does not make delivery synchronous, and polling cannot provide carrier-level certainty when the channel withholds a terminal event. It is not suitable for a few-second emergency alert, regulated retention without a separately managed archive, or high-volume marketing where consent, suppression, and throughput need specialized controls. Stick with a provider-native event stream or a dedicated compliance archive when those constraints dominate. I'm not sure every sender's retention semantics match your contract, so verify them in writing before promising a delivery history.

Email notices also carry legal obligations. Keep unsubscribe and sender identity rules separate from operational receipts; the FTC's CAN-SPAM guidance is a useful baseline for commercial mail, while OTP-like flows need rate limits, single-use tokens, and expiration as described by OWASP. These standards shape the policy layer even when the transport is swapped.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
- https://www.rfc-editor.org/rfc/rfc7231
