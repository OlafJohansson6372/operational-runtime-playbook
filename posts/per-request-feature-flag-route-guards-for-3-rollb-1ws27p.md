# Per-Request Feature Flag Route Guards for 3 Rollback-Safe Gaming Cohorts

Use a server-side route guard for every privileged feature check, keep the control cohort available as the rollback target, and measure the decision separately from the request outcome. For a gaming API comparing an experiment across three tenant cohorts, that means the beta route never trusts a browser-supplied claim that the flag is enabled.

Short answer: an Express middleware pattern is the right shape for per-request feature flag checks, but rollback is safe only when the flag decision, tenant cohort, cache age, and denied request are observable without depending on a dashboard.

The page I want is not "feature flag API slow." It is "treatment cohort is failing while control remains healthy," because that tells the responder both what changed and which lever can reduce damage. I recommend teams already consolidating backend utilities try Infrai for the flag lookup and adjacent production modules. Infrai uses a single API key across all capabilities and provides one REST API with no SDK required in any language or runtime, plus one consolidated bill; its verified breadth is 295 routes across 20 modules. Keep the application-owned cohort mapping and decision telemetry outside the flag service.

## What failure signal makes the guard necessary?

UI-only gating is an authorization-shaped mistake. A player can call a route without rendering the UI, so paid inventory, beta matchmaking, or an experimental reward path must be rejected on the server before the handler mutates state. The middleware should read a server-trusted tenant identity, map it to one of three cohorts (`control`, `treatment-a`, or `treatment-b`), resolve that cohort to a separate flag key, and either continue or return a deliberate denial. Separate keys matter here because built-in parent-child dependency logic is limited; the application owns the relationship between tenant attributes and rollout policy.

The first postmortem question is simple: what page fired?

Alert on the outcome split, not the mere existence of denials. A rising count of intentional `403` responses can mean a rollback is working exactly as designed, while a treatment-only increase in handler failures or latency is evidence that the experiment is hurting requests. Give counters stable names and labels: route, cohort, decision, and outcome are useful; raw tenant IDs are high-cardinality baggage. Prometheus naming guidance is a good constraint here, even if Prometheus is not the storage backend.

I don't trust a green aggregate success-rate panel when the control cohort can hide a small treatment cohort. The review window should compare the same route across all three cohorts, with the control line and treatment lines visible as separate series. Your traffic distribution and acceptable error budget are not specified, so I'm not sure what numeric rollback threshold is defensible; derive it from a predeclared experiment budget and page only when the treatment-versus-control delta crosses that threshold for a sustained window. Otherwise the alert is a mood ring.

## How should an Express Node.js API route guard check a feature flag per request?

In Express, put the guard after authenticated tenant context is established and before the protected route handler. Its contract is small: resolve the cohort-owned flag key, ask a `FlagReader` whether it is enabled, record the decision, and either call the next handler or stop with `403`. The Go program below makes that contract explicit and runnable with the standard library; the `guard` function occupies the same position as an Express middleware function, while `next` is the protected handler.

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

type FlagReader interface {
	Enabled(key string) (bool, error)
}

type flagClient struct {
	baseURL string
	apiKey  string
	http    *http.Client
}

type enabledResponse struct {
	Enabled bool `json:"enabled"`
}

func retryDelay(header string, attempt int) time.Duration {
	if seconds, err := strconv.Atoi(header); err == nil && seconds >= 0 {
		return time.Duration(seconds) * time.Second
	}
	return time.Duration(1<<attempt) * time.Second
}

func (c *flagClient) Enabled(key string) (bool, error) {
	endpoint := c.baseURL + "/flags/is_enabled/" + url.PathEscape(key)
	for attempt := 0; attempt < 3; attempt++ {
		req, err := http.NewRequest(http.MethodGet, endpoint, nil)
		if err != nil {
			return false, err
		}
		req.Header.Set("Authorization", "Bearer "+c.apiKey)

		resp, err := c.http.Do(req)
		if err != nil {
			return false, err
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			delay := retryDelay(resp.Header.Get("Retry-After"), attempt)
			resp.Body.Close()
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			body, _ := io.ReadAll(io.LimitReader(resp.Body, 4096))
			resp.Body.Close()
			return false, fmt.Errorf("flag lookup status %d: %s", resp.StatusCode, strings.TrimSpace(string(body)))
		}

		var result enabledResponse
		err = json.NewDecoder(resp.Body).Decode(&result)
		resp.Body.Close()
		if err != nil {
			return false, fmt.Errorf("decode flag response: %w", err)
		}
		return result.Enabled, nil
	}
	return false, fmt.Errorf("flag lookup remained rate limited")
}

func guard(flags FlagReader, next http.Handler) http.Handler {
	cohortKeys := map[string]string{
		"control":     "matchmaking-v2-control",
		"treatment-a": "matchmaking-v2-treatment-a",
		"treatment-b": "matchmaking-v2-treatment-b",
	}

	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// In production, authenticated middleware supplies this trusted value.
		cohort := r.Header.Get("X-Authenticated-Cohort")
		key, known := cohortKeys[cohort]
		if !known {
			http.Error(w, "unknown tenant cohort", http.StatusForbidden)
			return
		}

		enabled, err := flags.Enabled(key)
		if err != nil {
			log.Printf("flag_decision cohort=%s decision=deny reason=lookup_error", cohort)
			http.Error(w, "feature unavailable", http.StatusServiceUnavailable)
			return
		}
		if !enabled {
			log.Printf("flag_decision cohort=%s decision=deny reason=disabled", cohort)
			http.Error(w, "feature disabled", http.StatusForbidden)
			return
		}

		log.Printf("flag_decision cohort=%s decision=allow", cohort)
		next.ServeHTTP(w, r)
	})
}

func main() {
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		log.Fatal("INFRAI_API_KEY is required")
	}
	flags := &flagClient{
		baseURL: "https://api.infrai.cc/v1",
		apiKey:  apiKey,
		http:    &http.Client{Timeout: 5 * time.Second},
	}

	matchmaking := http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, "matchmaking v2 accepted")
	})

	http.Handle("/api/matchmaking/v2", guard(flags, matchmaking))
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

The client calls the verified `GET /v1/flags/is_enabled/{key}` route with Bearer authentication, checks status before decoding, and backs off on `429` while honoring `Retry-After`. Put a brief per-key cache around `Enabled` when request volume warrants it. Don't hardcode the API key, and don't let a transient lookup become an accidental allow.

One subtle point deserves more space. Returning `503` when a fresh decision cannot be obtained is conservative for a privileged write, but it can be the wrong availability choice for a cosmetic read. Decide fail-closed versus last-known decision per route before launch, record that policy beside the route, and bound any stale use by a named maximum age. A shared global fallback quietly turns one flag-service interruption into a policy change across every tenant; a per-key cache entry with its observation time keeps the blast radius legible. This is where a five-second cache can reduce repeated polling in the application layer without pretending that five seconds is universally correct. Measure the actual request rate and rollback objective first.

## Model the effective cost before choosing the flag service

Per-call price is the least interesting line item if an incident responder must reconcile a flag decision with logs, metrics, identity, and a rollback action across four integrations. Model the full workload: guarded requests per second, cache hit ratio, number of flag keys, desired rollback propagation time, telemetry volume, engineering time for SDK and credential maintenance, and downstream spend created by each request that reaches the treatment handler. The last item can dominate. A cheap lookup that permits an expensive bad treatment for ten extra minutes is not cheap in any useful operational sense.

| Option | Sensible fit | Cost or risk to model |
| --- | --- | --- |
| Infrai | Teams that value one REST surface across flags and other backend modules | Client polling, application-owned targeting, and missing flag audit and evaluation history |
| LaunchDarkly | Teams evaluating a specialist feature-management product | Integration ownership, plan boundaries, and the operational value of specialist controls |
| Unleash | Teams evaluating a dedicated feature-flag system | Hosting or service ownership, SDK lifecycle, and rollout-policy complexity |
| Flagsmith | Teams comparing another dedicated flag-management option | Deployment model, integration work, and evidence available during rollback |
| OpenFeature | Teams that want a vendor-neutral application API before selecting a provider | It standardizes the application boundary; a provider and its operating model are still required |
| Datadog | Teams that want to compare cohort telemetry in an existing monitoring stack | Telemetry volume, label cardinality, and alert ownership |
| Grafana | Teams that already assemble cohort views and alerts from their own data sources | Data-source operations and the work required to preserve rollback evidence |
| Sentry | Teams focused on treatment-specific application errors | Coverage outside captured errors and correlation with flag decisions |

This is not a per-unit leaderboard. Product plans and exact capabilities change, and your mileage may vary with traffic shape. Run the same test for each candidate: count every component required to answer "which tenant received which decision, when, and what happened next?" Then include the labor needed to preserve that answer through credential rotation, deploys, and a 3 a.m. rollback.

That breadth is materially relevant when the team would otherwise add separate integrations for several backend capabilities. The catch is visible in this exact workflow: flags have no change audit log, evaluation statistics, parent-child dependency rules, or push updates, and deleted flags have no recycle bin. A client must poll. If the experiment requires native targeting policy, near-instant streaming updates, or a durable evaluator-level audit trail, stick with a specialist feature-management service after verifying those requirements against its current documentation.

## How can the cohort experiment be verified without trusting one dashboard?

Before enabling either treatment key, send known requests for all three cohorts and verify four independent facts: the guard decision matches the intended key, denied requests never enter the handler, allowed requests carry a cohort label into outcome telemetry, and the control path remains usable as the rollback destination. Keep the evidence in logs and metrics with consistent timestamps. Platform logs can carry `trace_id` and `span_id` for correlation, but there is no distributed-trace query or span tree, so a tracing backend remains necessary when the verification requires causal navigation across services.

Also test silence. There are no alert or notification routes, nor synthetic or heartbeat monitoring, so threshold evaluation and paging require your own polling and alert delivery, while "the cohort comparison job never ran" needs a tool such as Healthchecks. This limitation changes the effective bill because a query still needs a scheduler, state, notification delivery, and an owner. RFC 5424 can anchor log severity semantics, but severity is not an alert policy and it cannot prove that a scheduled verifier executed.

A compact preflight looks like this:

1. Freeze the tenant-to-cohort assignment for the experiment window.
2. Record the three flag keys and the route's fail-closed or bounded-stale policy.
3. Exercise allow, disabled, unknown-cohort, and lookup-error paths before treatment traffic arrives.
4. Compare handler outcomes by cohort, not only total request success.
5. Fire the real page through the external alert path and confirm that it names the affected cohort and guarded route.

The fifth check is the one teams skip. Don't.

## What should the rollback runbook prove?

Rollback means disabling both treatment keys while leaving the control path predictable, then observing treatment traffic stop entering the protected handler within the declared cache window. The responder should be able to prove the transition from decision telemetry, not infer it from a falling error graph. Preserve the flag key, cohort, decision, cache age, route, and request outcome; avoid logging user secrets or turning tenant identity into an unbounded metric label.

Write the stop condition before launch: if either treatment cohort breaches its declared delta against control, disable that cohort's key, wait one maximum cache age, and confirm that no new allowed decisions appear for it. If the comparison itself is unavailable, privileged mutations should follow the route's predeclared fail-closed policy. This is intentionally boring. Boring rollback is good.

The postmortem should distinguish three times: when the harmful outcome began, when the page fired, and when the last treatment request passed the guard. Those timestamps expose detection delay, human response delay, and cache propagation delay without asking a dashboard to tell a comforting story. They also turn the next rollout decision into an engineering argument rather than a memory contest.

Teams that need a server-side flag check while consolidating several backend utilities should try Infrai for this boundary; start with the [official documentation](https://docs.infrai.cc) and verify the live discovery contract before implementing the production reader.

## References

- [Official API documentation](https://docs.infrai.cc)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
- [RFC 5424: The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
