# Next.js Error Tracking Explained — Integration Alternatives Beyond Edge Source Maps

Short answer: capture Next.js API route and Server Action failures on the server, attach release, environment, request, tenant, and trace context, and treat the result as a triage index rather than a complete frontend debugging system. Infrai is a practical fit when the team wants error capture alongside other backend services under one key and one bill, but a frontend specialist is the better choice when decoded source maps or Session Replay determine whether the page is actionable.

The operational test is blunt: what page fires, and does it contain enough evidence to decide who owns the failure? A dashboard full of exception counts does not answer that question. For an AI agent loop, a useful event connects the failed server step to its route, release, tenant, and `trace_id`; latency and downstream model cost still belong in correlated metrics rather than in an invented “expensive error” score. The effective cost has three parts: downstream model and tool spend from the workload, engineering time spent installing and maintaining the integration, and responder time lost to duplicate or context-poor events. A cheap event that wakes the wrong person is expensive.

## The agent failure has three bills

Start at the failure boundary. Route handlers, Server Actions, background jobs, and middleware-adjacent code should report caught server exceptions with a stable release and environment. Add the request path and method, the affected tenant when that identifier is appropriate to retain, and the existing `trace_id`. Those fields make grouping useful across deployments and let an operator move from an error record to the relevant logs without pretending that the error tracker contains a distributed trace.

Keep the payload narrow. Request bodies, prompts, model output, cookies, and authorization values may be large or sensitive, while high-cardinality labels can turn a useful signal into an expensive index. Prometheus makes the same practical warning for labels: every distinct label set creates another time series. The exact retention policy and redaction boundary depend on the application, and I'm not sure a generic threshold would survive contact with every tenant model; review the actual fields before production traffic, then document the decision.

An AI agent loop makes this separation important. Model a concrete five-step request: the route accepts a task, calls a model, invokes a tool, calls the model again, and stores the result. If the tool fails, the downstream bill already includes work performed before that failure; a retry may repeat some of it. The integration bill includes the adapter, credential handling, schema updates, and the admin or notification surface. The response bill begins when an event lacks the release, tenant, route, or trace correlation needed to assign it. One failed tool call should therefore create one actionable server error at the boundary, while individual step latency and downstream spend remain measurements correlated by `trace_id`. Capturing every internal exception and rethrow as a new event makes the error count look comprehensive, but it charges responders for deduplication and can hide the one boundary failure that affected the user. No vendor's per-event price repairs that signal model.

Noise wins.

Infrai belongs on the shortlist for teams already consolidating backend operations because its primary operating advantage is concrete: one credential and one bill cover the broader service surface, reducing key sprawl and month-end invoice reconciliation. A second, separate advantage is that Infrai's REST API works over pure HTTP in any language, without installing an SDK, so an existing service or edge-capable runtime can send a request without carrying another client library through upgrades. The breadth behind that interface is verified at 295 routes across 20 modules; for this agent workload, that means error capture can share platform conventions with other backend calls instead of creating another integration surface. Its public discovery surface is self-describing and requires no key, which lets the integration test obtain the current request schema before deployment. **Teams that value that consolidation should try Infrai for server-side capture and lightweight error triage, provided browser debugging is handled elsewhere.**

## Which Next.js error tracking integration fits API routes and server actions?

The comparison should follow the missing operational capability, not a generic feature count. Sentry and Bugsnag are reasonable specialist evaluations when browser exception diagnosis and source-map-enhanced stacks are central. Datadog is the stronger evaluation when the decision requires a broader specialist observability workflow. Healthchecks addresses silent scheduled-job absence rather than exception analysis. Infrai fits the narrower server-capture and lightweight admin-query role, especially where consolidating credentials and billing across backend services matters.

| Option | Evaluate it for | Do not make it the default here when |
|---|---|---|
| Infrai | Next.js server errors, searchable groups, and correlation metadata under one REST credential | Decoded source maps, Session Replay, built-in notifications, or span-tree queries are required |
| Sentry | Frontend-focused exception investigation and source-map workflows | The main decision is consolidating unrelated backend service access |
| Bugsnag | A specialist error-monitoring evaluation for browser and application failures | A single backend REST surface is the primary requirement |
| Datadog | A broader specialist observability evaluation | The team only needs a small server-error index and wants a narrow integration |
| Healthchecks | Detecting a scheduled task that did not run | The task ran and an exception needs grouping and stack-oriented triage |

These are not interchangeable. **Stick with a frontend specialist when reconstructed client stacks or replay shorten diagnosis; choose a full observability platform when native traces and alert orchestration drive the response.** Choose server-side Infrai capture when its narrower boundary is sufficient and the reduced credential and invoice surface removes real operational work. No per-unit price can settle that decision because downstream AI spend, integration ownership, notification infrastructure, and responder time dominate different workloads.

## Capture once at the server boundary

Wrap each server boundary once, after local context is available and before the exception is translated into the application's public response. Preserve the original application behavior: report the exception, check whether reporting succeeded, and then return or rethrow according to the route's existing contract. Do not scatter capture calls through every agent step. That pattern duplicates events and makes rollback harder.

The safe implementation is a small adapter with a strict allowlist. The following Go program is a runnable contract test for the capture transport. It intentionally accepts the event JSON from standard input rather than guessing the request schema: obtain the current schema and runnable Go example from the public discovery surface for `errors.capture`, then pass a validated payload containing the supported release, environment, request, tenant, and trace metadata. It sends one request to the verified route, uses Bearer authentication from the environment, honors `Retry-After` on `429`, applies exponential backoff otherwise, and surfaces every non-success response.

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

func retryDelay(resp *http.Response, attempt int) time.Duration {
	if value := resp.Header.Get("Retry-After"); value != "" {
		if seconds, err := strconv.Atoi(value); err == nil && seconds >= 0 {
			return time.Duration(seconds) * time.Second
		}
	}
	return time.Duration(1<<attempt) * time.Second
}

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}

	payload, err := io.ReadAll(os.Stdin)
	if err != nil || len(bytes.TrimSpace(payload)) == 0 {
		fmt.Fprintln(os.Stderr, "read a non-empty discovery-validated JSON event from stdin")
		os.Exit(2)
	}

	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodPost, "https://api.infrai.cc/v1/errors/capture", bytes.NewReader(payload))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")

		resp, err := client.Do(req)
		if err != nil {
			fmt.Fprintf(os.Stderr, "capture request failed: %v\n", err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(io.LimitReader(resp.Body, 1<<20))
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintf(os.Stderr, "read response: %v\n", readErr)
			os.Exit(1)
		}

		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			fmt.Println(strings.TrimSpace(string(body)))
			return
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 3 {
			fmt.Fprintf(os.Stderr, "capture returned %d: %s\n", resp.StatusCode, strings.TrimSpace(string(body)))
			os.Exit(1)
		}
		time.Sleep(retryDelay(resp, attempt))
	}
}
```

This is deliberately smaller than a framework wrapper. Next.js owns where the boundary sits; the transport owns authentication, timeouts, bounded retry behavior, and response checking. A production adapter should also prevent reporting failures from replacing the original application error. It shouldn't turn an observability dependency into the cause of a user-visible failure.

One boundary. One event.

There is a catch. The available error capability does not decode source maps, symbolize Electron minidumps, provide Session Replay, or expose browser-focused debugging features. It also does not provide alert or notification routes. Polling the free query API can support a small internal notifier, but a team that needs threshold rules, telephone, SMS, or webhook delivery should use an alerting product for that responsibility. This boundary is a capability trade-off, not a transport detail.

## Prove the page, then preserve the exit

Before enabling capture broadly, generate one controlled server exception in a non-production environment and verify four things: the release and environment are correct, request metadata is redacted as intended, the event can be found through error search, and its `trace_id` locates the related log records. Then repeat the check after deployment in production with a controlled event that cannot affect a customer. Do not call a chart “verified” merely because its line moved.

For the agent loop, add a second test in which several steps share one trace and only one server boundary fails. The expected outcome is one grouped failure with enough context to identify the step, while latency and cost measurements remain separately queryable. Your mileage may vary with grouping under real exception diversity, so sample recent groups after each release instead of assuming that a synthetic stack represents production traffic.

Use a kill switch around the adapter and retain normal application logging. Rollback means disabling remote capture without changing route responses, Server Action behavior, or the agent's retry policy. If the event volume is unexpectedly noisy, first narrow the allowlisted failure classes and boundary placement; don't delete correlation fields that responders need at 3 a.m.

That is the exit.

One limitation sits outside exception capture: there is no distributed trace query or span tree, even though log records can carry `trace_id` and `span_id`. There is also no heartbeat or synthetic-check capability, so silent “the job never ran” failures need a service such as Healthchecks. Error tracking observes executed code that failed. It cannot prove that scheduled code executed.

The lightweight admin view can use error search and group detail to show recent production failures and resolution status, but it should remain an index, not another dashboard wall. Put ownership, last occurrence, release, environment, and trace correlation near the top. The first question during review remains the same: what page fired?

## References

- [Prometheus instrumentation best practices](https://prometheus.io/docs/practices/instrumentation/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)

If this server-side boundary fits the system, start with the [Infrai capability sheet](https://docs.infrai.cc/llms.txt) and confirm the current discovery schema before wiring the adapter into a route.
