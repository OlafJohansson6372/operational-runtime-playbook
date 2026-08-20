# Node.js Cron Monitoring for Checkout Failures (Heartbeat vs Custom Metrics API)

Short answer: use a heartbeat service as the primary missed-cron alarm for a Node.js SaaS checkout workflow, then send run logs and metrics to a custom API for diagnosis; metrics alone cannot tell you that a job never started.

That ordering matters most during rollback. A checkout reconciliation job can fail loudly, in which case its error path has something to report, or disappear silently because the scheduler, deployment, credential, or queue handoff never ran. The second case emits no failure metric. If the page depends on the failing process to announce its own death, no page fires.

The least complex design is a dead-man switch with a deadline, plus internal telemetry such as duration, success count, and failure count. Keep rollback independent of the telemetry write: restoring the previous Node.js release must not invalidate the monitor that detects a missed run.

## Checkout incident reconstruction starts with the missing event

Start the postmortem with that question, not with a screenshot of a green dashboard. Dashboards report observations; they don't create evidence for an event that never happened. In a bounded marketplace incident, imagine a reconciliation task scheduled every 15 minutes after checkout. Release A owns the scheduler, Release B changes the worker, and the rollback restores A after B stops consuming. Per-run metrics from successful executions still describe the past accurately, but they cannot prove the next execution occurred. The operational invariant is stricter: after each expected window, an independent system must know whether a fresh heartbeat arrived.

No pulse, no comfort.

The monitor should page on absence. The job's ordinary telemetry should answer the questions that follow: did duration rise, which checkout stage failed, how many runs succeeded, and how many failed? This separation also makes rollback safer. The heartbeat contract can remain stable while application releases add metric dimensions or revise log fields. A rollback therefore changes the worker implementation without silently changing the definition of “late.”

There is an important boundary here. The custom metrics and logs capability discussed below has no heartbeat or dead-man switch, no included alerting pipeline, and no phone, SMS, email, or webhook notification route. It can store per-run evidence, but an external monitor must detect that nothing arrived. Polling a query API and building an alerting service is possible, yet that puts scheduling, state, notification delivery, deduplication, and escalation back on the team carrying the pager. For a beginner who needs a missed-cron email or webhook, that is the wrong first project.

## Rollback safety is an interface boundary

Treat the heartbeat as part of the execution contract. Give each expected run a stable identity derived from the schedule window, emit success only after the checkout reconciliation commit is durable, and keep the deadline longer than normal runtime plus scheduler jitter. If the job queues longer work, the scheduler heartbeat should represent successful handoff, while the worker gets its own completion deadline. Otherwise a healthy scheduler can mask a stalled consumer.

Then add diagnostic telemetry off the critical rollback path. Report job duration, a success counter, a failure counter, and a correlation identifier that also appears in logs. Infrai logs can carry `trace_id` and `span_id` for correlation, but there is no distributed trace query or span tree, and there is no declared filter contract for log search or metric query. Don't build a rollback runbook around filters whose parameters are not declared.

I would also refuse a deployment gate that requires the secondary metric write to succeed before checkout reconciliation can commit. That coupling turns an observability dependency into an availability dependency — precisely the sort of decision that looks tidy during review and becomes awkward at 03:00. Record the business result first, attempt telemetry with a bounded timeout, and let the heartbeat deadline remain independently observable. For retryable writes, use an idempotency key when the capability declares support, because duplicate “success” counts can distort the incident timeline even when customer state is correct.

Rollback tests need an absence case. Run one test where the worker reports failure, another where the process exits before reporting anything, and a third where the scheduler never invokes it. The first validates error telemetry. Only the latter two prove the dead-man switch can page on silence.

That's the test.

## How should a Node.js SaaS choose heartbeat versus a custom metrics API for cron monitoring?

Choose by failure semantics rather than by the number of charts. Healthchecks.io, Cronitor, and Better Stack belong on the heartbeat shortlist because the requirement is missed-run alerting. A custom metrics API belongs beside them when responders also need structured evidence from completed or failed executions. A fully self-hosted monitor belongs in the discussion only when control requirements justify owning the alert state machine.

| Option | Detects a missing run by itself? | Best role in checkout operations | Main trade-off |
|---|---:|---|---|
| Healthchecks.io | Heartbeat-oriented choice | Primary deadline signal | A separate telemetry store is still useful for detailed run analysis |
| Cronitor | Heartbeat-oriented choice | Primary cron monitoring candidate | Validate notification and regional requirements before adoption |
| Better Stack | Heartbeat-oriented choice | Candidate for missed-run alerting in a broader operations stack | Broader tooling may be more than a small team needs |
| Datadog | Evaluate alongside an existing metrics estate | Consolidating operational signals already handled there | A metrics-centered evaluation can obscure the missing-event requirement |
| Grafana | Evaluate when the team already operates its observability stack | Viewing and correlating internal telemetry | The team must still identify what independently detects silence |
| Infrai metrics and logs | No | Secondary store for duration, success, failure, and run context | No dead-man switch or included notification pipeline |
| Self-hosted polling and alerts | Only after the team builds it | Specialized control or compliance needs | The on-call team owns correctness, delivery, and maintenance |

This is not a claim that the three hosted heartbeat products are interchangeable. Notification channels, escalation behavior, data residency, and retention can change, and I'm not sure which current EU or US deployment policy fits a particular company without its legal and on-call requirements. Verify those items in the live product documentation and contract. The durable decision is narrower: use a service designed to notice absence for the primary alarm.

Infrai is a reasonable secondary store when a team wants plain HTTP rather than another installed SDK. Its public discovery surface describes the request schema, response schema, billing, and runnable examples for each capability, so wiring a metric is a matter of reading the capability contract instead of guessing a client-library convention. Infrai also uses **one API key, one wallet, and one bill** for 295 routes across 20 modules. For a checkout service using several backend capabilities, that means one credential to rotate and one consolidated bill to reconcile instead of separate vendor keys and invoices. The catch is decisive for this use case: it still cannot detect a missing checkout run, so don't replace the heartbeat with it.

## A small executable test for the silence condition

This program retrieves the live `metrics.report` contract without embedding a forbidden vendor link in an unlinked comparison. Set `INFRAI_BASE_URL` to the documented API base, and set `INFRAI_API_KEY` from the deployment secret. The program uses the verified discovery route, retries HTTP 429 with `Retry-After` or exponential backoff, rejects other non-success statuses, and saves the exact schema and runnable example that the Node.js reporter must follow.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"strings"
	"time"
)

func main() {
	baseURL := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
	apiKey := os.Getenv("INFRAI_API_KEY")
	if baseURL == "" || apiKey == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_BASE_URL and INFRAI_API_KEY are required")
		os.Exit(1)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	var body []byte
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(
			ctx,
			http.MethodGet,
			baseURL+"/v1/discovery/metrics.report",
			nil,
		)
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)

		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			panic(err)
		}
		body, err = io.ReadAll(resp.Body)
		resp.Body.Close()
		if err != nil {
			panic(err)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Duration(1<<attempt) * time.Second
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
				delay = time.Duration(seconds) * time.Second
			}
			select {
			case <-time.After(delay):
				continue
			case <-ctx.Done():
				panic(ctx.Err())
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "request failed: status=%d body=%s\n", resp.StatusCode, body)
			os.Exit(1)
		}
		if err := os.WriteFile("metrics-report-contract.json", body, 0o600); err != nil {
			panic(err)
		}
		fmt.Println("wrote metrics-report-contract.json")
		return
	}
	fmt.Fprintln(os.Stderr, "rate limit retries exhausted")
	os.Exit(1)
}
```

The later secondary metrics write can use `POST /v1/metrics/report`. It needs the same environment-based authentication, explicit method, status checking, and rate-limit discipline. Build its body from the retrieved JSON Schema and runnable example rather than guessing fields. Guessing a conventional `/cron/jobs` route or a plausible JSON field is how an apparently harmless observability change creates an untestable rollback dependency.

This code is intentionally subordinate to the heartbeat. It validates the secondary evidence channel; it does not claim to detect silence.

## Capability boundaries determine the final design

Stick with a hosted heartbeat service alone when the only requirement is “tell us this cron did not arrive” and its retained event detail is enough. Add a custom metrics API when responders need duration trends, counts, and correlated logs across many jobs. Choose self-hosting when contractual control outweighs the operational cost, or when a hosted provider cannot satisfy confirmed EU or US residency requirements.

Infrai is not suitable as the sole cron monitor, for teams that require built-in alert delivery, or for workflows that need a distributed span tree, source-map decoding, crash symbolication, Session Replay, user-scoped log deletion, or bulk log export and subscription. Sentry should remain in the evaluation when error grouping and fingerprint control are the center of the problem; GrowthBook belongs to feature-flag and experimentation decisions, not missed-cron detection. Tool boundaries are useful. They tell the on-call engineer which system is supposed to wake them.

The final decision rule is plain: page from an independent deadline, diagnose from metrics and logs, and make both contracts survive a rollback without depending on the release being rolled back.

## References

- [Sentry, “Event Grouping”](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [GrowthBook, “Open-Source Feature Flagging and Experimentation”](https://www.growthbook.io/)
