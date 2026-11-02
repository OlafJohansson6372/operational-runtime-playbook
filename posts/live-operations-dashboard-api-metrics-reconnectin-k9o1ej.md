# Live Operations Dashboard API Metrics: Reconnecting Game Editor Cursors Beyond Polling

Short answer: for a game editor's collaborative cursors, use a long-lived bidirectional stream for live movement, but treat reconnect and backfill as a separate state-recovery protocol. Polling can keep a small, low-stakes panel fresh; it is a poor definition of correctness for ephemeral cursor events. The invariant is simple: after a client reconnects, its view must converge on the server's current cursor snapshot, even if some movement events were missed.

The operational constraint changes the design. A cursor is disposable, while the editor revision and the cursor's latest position are not interchangeable. If a dashboard shows a stale cursor for 20 seconds, the damage is mostly confusion. If the editor applies a stale revision as if it were current, two people can appear to be editing the same object while looking at different state. I would page on the latter, not on a temporary drop in movement events.

I have been woken by alerts that meant nothing and missed the one that mattered. That is why I keep the cursor freshness signal separate from the editor-revision signal.

The recovery result is part of the product contract.

## The incident boundary: what did the page actually mean?

Consider a bounded production incident in a collaborative level editor. A player-support engineer opens a map, two designers join, and the connection carrying cursor updates is interrupted during a network change. The client reconnects, resumes receiving movement events, and still displays one designer at the old location. The visible symptom is a frozen cursor. The real question is narrower: did the client miss disposable deltas, or did it miss the authoritative state required to interpret later edits?

That distinction determines the recovery path. A movement event can carry a sequence number and a position, but replaying every old movement is usually wasted work; the useful result is the latest position for each active editor session. An editor revision needs stronger treatment. It should be identified separately, checked against the client's known revision, and recovered through a snapshot or a bounded event backfill before the client resumes normal interaction.

I distrust dashboards that report only an open socket count. A socket can be open while its consumer is stuck, its sequence is advancing nowhere, or its authorization has expired. The useful signals are the last received sequence per client, the age of the newest accepted cursor update, the number of recovery attempts, and the mismatch between the client's revision and the server's revision. At 3am, those answer what page fired.

## How should a live ops dashboard API stream metrics after disconnects?

Use two paths with one convergence rule:

1. The live path sends cursor updates over a bidirectional stream. Each update includes an editor session identifier, a monotonically increasing sequence for that session, the current editor revision, and the cursor position.
2. The recovery path starts with a fresh authorization check, then asks for either a snapshot or events after the client's last known sequence. The server decides whether that history is still available.
3. The client applies a recovery result atomically, records the resulting sequence, and only then accepts live updates whose sequence is newer.

The server should be allowed to return a snapshot when the requested history is gone. That is not a failure of the protocol; it is the protocol refusing to pretend that a partial replay is complete. A client that receives sequence 418 after its last accepted sequence was 400 must not silently infer that 401 through 417 were irrelevant to editor state. It can discard old cursor positions, but it needs an explicit rule for the revision they were attached to.

Here is the shape of the client-side gate in Go. The transport is deliberately generic: it could sit behind a WebSocket connection or another bidirectional session, and the recovery endpoint is an application boundary rather than a product-specific API.

```go
type CursorUpdate struct {
	SessionID string
	Seq       uint64
	Revision  uint64
	X         int
	Y         int
}

type CursorState struct {
	Revision uint64
	Seq      uint64
	X        int
	Y        int
}

func acceptCursor(state CursorState, update CursorUpdate) (CursorState, bool) {
	if update.Seq <= state.Seq {
		return state, false
	}
	if update.Revision < state.Revision {
		return state, false
	}
	return CursorState{
		Revision: update.Revision,
		Seq:      update.Seq,
		X:        update.X,
		Y:        update.Y,
	}, true
}
```

The exact comparison is a policy choice. In an editor where cursor movement is cosmetic, revision equality may be enough and a newer revision can trigger a separate editor-state check. In an editor that binds cursor position to an object transform, a newer cursor revision should be rejected until the corresponding editor state is present. The important part is making that boundary explicit instead of letting arrival order define truth.

## Why is polling attractive until the first gap?

Polling is easy to reason about in a small dashboard: every interval asks for a current snapshot, and the next response replaces the previous one. It also gives operators a natural request rate to observe. The cost is temporal rather than financial: a poll interval creates a blind window, synchronized clients can create request bursts, and a response says little about what happened between snapshots. For cursor movement, those missing intermediate positions are normally harmless. For a recovery cursor that must be paired with an editor revision, silent gaps are harder to diagnose.

A stream has a different failure shape. It can deliver changes promptly and avoid repeated empty reads, but it requires explicit handling for disconnects, backpressure, authentication expiry, duplicate delivery, and reconnect storms. The stream is not the reliability mechanism by itself. Sequence numbers, bounded retention, snapshot fallback, and idempotent application are the mechanism.

The trade-off is real. This approach is a poor fit for a small read-only dashboard whose freshness target allows a periodic snapshot; there, a stream adds connection lifecycle and recovery work without improving the user-visible result. Polling is also easier to operate when the source has no event identity or retained history. Choose the stream only when the workflow benefits from prompt changes and the team is willing to own a recovery contract.

The same distinction applies to common implementations. WebSocket gives an application a bidirectional channel, but the application still defines message ordering, authorization renewal, and recovery. Server-sent events provide a one-way event stream and can fit a read-only operations view, but they do not replace a bidirectional editor protocol. WebRTC data channels can carry peer-to-peer data, while the W3C specification leaves the application responsible for deciding how that data maps to authoritative editor state and how peers recover after membership or connectivity changes. These are different transport capabilities, not interchangeable delivery guarantees.

## What should the reconnect test prove?

The useful test is not “does the socket reconnect?” It is “does the client converge when the outage happens at the least convenient point?” I would make the harness pause delivery after sequence 400, move the cursor through 401 to 417, close the stream, and reconnect with 400 as the last accepted sequence. The server should return either the retained range or a snapshot marked with its resulting sequence and revision. After recovery, an old duplicate must not move the cursor backward.

Add a second case where the editor revision changes during the pause. The cursor response alone cannot prove that the client has the corresponding editor state. The client should hold interaction, recover the revision through its state path, and resume only after both views identify the same revision. That is slower than painting a position immediately. It is also much easier to explain during a postmortem.

A practical acceptance table looks like this:

| Condition | Client action | Operator signal |
| --- | --- | --- |
| Duplicate or older cursor sequence | Ignore it | Duplicate count, sampled by session |
| Short outage with retained history | Replay, then resume | Recovery duration and replay size |
| History unavailable | Apply authoritative snapshot | Snapshot fallback count |
| Revision mismatch | Pause editor interaction | Mismatch age and affected sessions |
| Repeated reconnects | Back off with jitter and reauthorize | Attempts per session and rejection reason |

Do not make every metric a page. A brief loss of cursor freshness is expected during a reconnect. A growing revision-mismatch age, a recovery loop, or a client that accepts updates without a known baseline deserves an alert because each points to a convergence failure rather than ordinary network noise.

## The decision rule

For a live ops dashboard that only displays aggregate health, polling a current snapshot remains a defensible choice when the freshness target tolerates its interval and the system does not need event history. For collaborative cursors in a game editor, use a stream for the interactive path and a snapshot-or-backfill protocol for recovery. Keep cursor presence ephemeral, keep editor revisions authoritative, and measure the gap between them.

The exception is a deliberately tiny system where a single periodic snapshot is the documented product behavior and no user action depends on event order. In that case, polling is simpler and may be the better operational decision. Once the UI promises immediate collaboration or a reconnect-safe editing session, the protocol needs a baseline and a convergence test.

The conclusion I would carry into a postmortem is narrow: transport selection is only the first decision. The production contract is what happens after the transport lies by omission. If the client can identify its last accepted state, request an honest recovery result, and reject data older than that result, a dropped connection becomes a bounded recovery event instead of a mystery cursor that keeps pointing at the wrong object.

## Sources

- W3C, WebRTC 1.0: https://www.w3.org/TR/webrtc/
- RFC 6455, The WebSocket Protocol: https://www.rfc-editor.org/rfc/rfc6455
- WHATWG, Server-sent events: https://html.spec.whatwg.org/multipage/server-sent-events.html
- MDN, WebSocket API: https://developer.mozilla.org/en-US/docs/Web/API/WebSocket
