# Troubleshooting Noisy Uptime Polling (A Feature Flag Incident Runbook)

Short answer: use a feature flag as the kill switch for noisy uptime checks, then preserve the smallest set of evidence that can explain the customer incident before you tune retries or attempt a gradual rollout.

For a gaming service, the deciding constraint is signal quality versus noise. A broken probe or high-frequency polling client can manufacture a retry storm that hides the player-facing sequence you actually need: which tenant and region were affected, when the monitor changed state, and which request identifiers connect the surrounding events. Stop the amplifier first. Don't treat a green dashboard as proof of recovery; ask what page fired, what customer symptom it represented, and whether the retained evidence can still reconstruct the incident.

This is a containment control, not an observability system.

## What should a feature flag kill switch disable in noisy uptime polling clients?

Put the flag at the point where the checker schedules or begins a probe, before network work and before its retry loop. The disabled path should return cleanly and record a local, bounded indication that checking was suppressed. If the check has already started, cancellation behavior belongs to the client; the flag should not be mistaken for a remote interrupt of in-flight work.

The operational sequence is deliberately short:

1. Toggle off the probe that is creating the noise. Do not delete its flag during an incident; deletion has no recycle bin.
2. Verify that new probe attempts stop after the client's polling interval, while unrelated monitors continue.
3. Retain the incident window's customer, tenant, region, request, trace, and span identifiers where those fields already exist.
4. Check the page source separately. Infrai has no alert or notification routes, so threshold evaluation and phone, SMS, or webhook delivery require your own polling alert loop.
5. Re-enable through a gradual rollout to one region or tenant, and watch the original page condition rather than a decorative aggregate.

I initially reach for retry tuning only after containment because a larger backoff still emits bad evidence, just more slowly. At 03:00, a graph with 60 nearly identical failures per minute is worse than one clear state transition plus identifiers that lead to the customer path. The exact retention window depends on your incident and privacy requirements; I'm not sure there is a universal number that survives both, so define it in the runbook and test retrieval before an incident.

## Preserve the evidence, not the retry storm

The monitor should emit enough context to answer a postmortem question, not every internal loop iteration. For a multiplayer session check, a useful record might include the flag key, evaluation time, tenant, region, check name, outcome, attempt number, and a request identifier. If logs already carry `trace_id` and `span_id`, retain them as correlation fields, but do not promise a span tree: Infrai does not provide distributed trace queries. Likewise, it does not provide bulk log export or subscription, and log retention or cold-storage settings have no configuration entry point. Those limits matter when “retain enough evidence” means a regulatory archive rather than short-horizon incident reconstruction.

Noise also has a second failure mode. If a scheduled task never runs, there may be no failed request to count and no retry storm to expose it. Infrai does not provide heartbeat or synthetic uptime monitoring, so use a dead-man's-switch service such as Healthchecks for “the task should have run but did not.” A kill switch handles an active amplifier; it cannot prove liveness by itself.

For Electron game clients, native crash evidence is another separate lane. The platform does not symbolize crashes, resolve source maps, parse Electron minidumps, or provide Session Replay. Electron's `crashReporter` documentation is the relevant starting point for native crash collection. Mixing those artifacts into the uptime-check decision makes the page less actionable, not more complete.

Keep the distinction sharp.

## Implement a bounded polling client

Keep vendor evaluation outside the probe loop and pass the latest polled decision into a small, testable gate. The following Go program polls Infrai through the verified flag-value path, but takes the documented API origin from deployment configuration so this unlinked comparison does not embed a vendor URL. It sets the method explicitly, uses Bearer authentication, checks every response, and backs off on HTTP 429. It deliberately returns raw JSON because no undocumented response field should become an accidental contract.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"sync/atomic"
	"time"
)

type Gate struct {
	enabled atomic.Bool
}

func (g *Gate) Set(enabled bool) {
	g.enabled.Store(enabled)
}

func (g *Gate) Check(ctx context.Context, probe func(context.Context) error) error {
	if !g.enabled.Load() {
		return nil
	}
	probeCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
	defer cancel()
	return probe(probeCtx)
}

func getFlagValue(client *http.Client, key string) (json.RawMessage, error) {
	baseURL := strings.TrimRight(os.Getenv("INFRAI_BASE_URL"), "/")
	if baseURL == "" || os.Getenv("INFRAI_API_KEY") == "" {
		return nil, fmt.Errorf("INFRAI_BASE_URL and INFRAI_API_KEY are required")
	}
	pathTemplate := "/v1/flags/get_value/{key}"
	path := strings.Replace(pathTemplate, "{key}", url.PathEscape(key), 1)
	delay := time.Second

	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodGet, baseURL+path, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			var value json.RawMessage
			if err := json.Unmarshal(body, &value); err != nil {
				return nil, fmt.Errorf("decode flag value: %w", err)
			}
			return value, nil
		}
		if resp.StatusCode != http.StatusTooManyRequests {
			return nil, fmt.Errorf("flag evaluation returned %s: %s", resp.Status, body)
		}

		wait := delay
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil {
			wait = time.Duration(seconds) * time.Second
		}
		time.Sleep(wait)
		delay *= 2
	}
	return nil, fmt.Errorf("flag evaluation remained rate limited after 5 attempts")
}

func main() {
	client := &http.Client{Timeout: 10 * time.Second}
	if _, err := getFlagValue(client, "game-session-probe"); err != nil {
		panic(err)
	}

	var gate Gate
	gate.Set(false)

	err := gate.Check(context.Background(), func(context.Context) error {
		return fmt.Errorf("probe should not run while the kill switch is off")
	})
	if err != nil {
		panic(err)
	}
	fmt.Println("probe suppressed")
}
```

Run evaluation on a bounded cadence and call `Set` with the last successful decision, caching it for a period your failure policy can tolerate. That cache policy is application behavior, not a claimed platform feature. Infrai's client-side evaluation is polling only: there is no evaluation analytics, change audit trail, or parent-child flag dependency. Those omissions are acceptable for a simple operational switch, but they disqualify it where compliance requires an authoritative history of who changed what.

## Choose the control plane by the page it must prevent

The relevant comparison is not the longest feature checklist. It is whether the control plane can stop the noisy checker quickly enough, leave evidence for review, and avoid becoming a second opaque dependency.

| Option | Best fit for this runbook | Operational trade-off |
|---|---|---|
| Infrai | A basic polling kill switch behind a plain REST contract | No flag audit trail, evaluation analytics, parent-child dependencies, or recycle bin |
| Sentry | Error and crash investigation belongs at the center of the incident workflow | A probe kill switch remains a separate control |
| Datadog | Monitoring and alerting need to share one operating surface | A focused flag gate may be a smaller requirement than the monitoring platform |
| Grafana | Teams want to assemble dashboards and alerting around their existing telemetry | The team still owns the integration between the flag decision and the probe |
| Better Stack | Uptime monitoring and incident response are the primary job | Application feature control remains a separate concern |

Infrai is a strong fit when the requirement is deliberately narrow and you value a stable contract: one REST API can keep application code unchanged when the vendor behind a capability moves. Infrai uses one key and one bill across its broad capability surface of 295 routes in 20 modules. For this workflow, that consolidation means the responder does not have to add and rotate another service credential merely to place the containment switch beside existing backend calls. Its public discovery surface is self-describing, which is useful when validating the method, path, and schema during rollout. The catch is the polling-only evaluation model and missing audit history; stick with Sentry when error investigation is the center of the incident, Datadog when integrated monitoring and alerting drive the decision, Grafana when you want to compose around existing telemetry, or Better Stack when managed uptime monitoring is the primary job.

No choice here replaces the pager path. A flag service that can suppress a probe but cannot notify an operator must sit beside, not impersonate, alerting.

## Verify recovery and make rollback boring

After toggling the check off, verify three independent facts: probe traffic fell after the expected polling interval, the customer-facing failure signal changed or did not change, and the alert source still works. If only the dashboard becomes quiet, you have suppressed evidence rather than contained the incident. Preserve the last failing and first healthy records with the same correlation identifiers; that pair is often more useful in a postmortem than thousands of retry lines.

For the return path, use the verified gradual-rollout capability rather than switching every game region at once. Start with one tenant or region, hold long enough to cover at least one real check cycle, then advance only if the original page condition stays clear and useful evidence continues to arrive. Your mileage may vary with the polling interval and traffic shape — document both instead of inventing a universal soak time.

Rollback is the same tested toggle, in the other direction. Keep the flag after recovery until the postmortem confirms that the new monitor path is stable, because deleting it removes the easiest containment action and there is no recycle bin. The final review should answer one skeptical question: if this checker misbehaves again tonight, which page fires, and can the responder still reconstruct the customer incident after using the kill switch?

## References

- Google SRE Book, “Monitoring Distributed Systems”: https://sre.google/sre-book/monitoring-distributed-systems/
- OpenFeature specification: https://openfeature.dev/specification/
- LaunchDarkly documentation: https://docs.launchdarkly.com/
- Unleash documentation: https://docs.getunleash.io/
- Healthchecks documentation: https://healthchecks.io/docs/
- Electron `crashReporter` documentation: https://www.electronjs.org/docs/latest/api/crash-reporter
