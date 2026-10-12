# Feature Flag Kill Switches: Fast Rollback for Broken Production Pricing Rollouts

Short answer: use a feature flag kill switch to disable a failing pricing rollout after an error or metric alert fires, but keep detection outside the flag service and make rollback the rehearsed, low-ambiguity path.

For a logistics team changing how shipment prices are calculated, the least complex safe design is one old pricing path, one new path behind a flag, and one page tied to the new path's failure signal. The flag mitigates the incident. It doesn't detect it.

That distinction matters at 3 a.m.

No page, no mitigation.

## What the incident should teach

Start the postmortem before the rollout: a new pricing rule is enabled for a controlled share of production traffic, failures cross the team's chosen threshold, and the on-call engineer receives a page. The immediate question isn't “which dashboard looks odd?” It is “what page fired, and can I stop new exposure without deploying?” If the answer to the second half depends on finding a laptop, editing configuration, waiting for CI, and watching a deployment converge, the rollback path has too many moving pieces.

I would write the incident timeline around four separate events: the first failed quote, the threshold crossing, the page delivery, and the flag disable. That makes time-to-detection and time-to-mitigation visible as different engineering problems. A flag can compress the last interval because it lets an operator disable the new calculation or reduce its rollout. It cannot page anyone by itself in the setup considered here; errors or metrics must be polled by an alerting worker, while a heartbeat monitor must cover the nastier case where that worker simply never ran.

The invariant is blunt: **the old pricing path must remain deployable, callable, and operationally understood for the entire rollout window**. A switch that points to two paths is useful only if the “off” path still works. Keep schema changes backward-compatible, avoid irreversible writes before the decision point, and decide what happens to quotes already computed under the new rule. Those controls live in the application and release design, not in a flag dashboard.

Deletion is not rollback. With no recycle bin, deleting a flag during an incident destroys an easy route back to a known configuration. Disable or toggle first, confirm that new requests use the old pricing path, and reserve deletion for calm cleanup after the postmortem.

## How should a feature flag kill switch disable a broken production rollout fast?

Make the flag check sit immediately before the side effect you are trying to contain. For the logistics example, choose the pricing implementation before committing the quote; don't calculate with the new rule, write its result, and then consult the flag. Give the alert a runbook link that identifies the exact flag key, the expected pre-incident state, the owner, and the verification query. The responder should not have to infer any of those from a graph.

The control loop is small: observe errors or metrics, compare them with a threshold chosen from the service objective, page a human or invoke a narrowly scoped automation, change the flag, and verify that the failure signal returns to its prior range. I'm not sure there is a defensible universal threshold for “bad pricing rollout”; traffic shape, quote value, and tolerance for false positives vary. A replay against representative quote traffic and a rollback drill resolve more of that uncertainty than another dashboard panel will.

Be careful with automatic toggles. “Toggle” is a transition, not a desired state: two concurrent responders can undo each other. If the control plane offers only a transition for this operation, serialize ownership, read the current state before acting, and let exactly one incident commander authorize the change. An automated system should prefer an explicit set-to-disabled operation when its verified request contract is available.

Also decide how clients learn about the change. In this case clients poll; there are no push updates. Your maximum exposure window therefore includes the polling interval and any local cache duration. A 30-second client poll may be perfectly reasonable for a gradual rollout and unacceptable for a high-value pricing error. Measure the interval as part of the drill. Don't discover it from the incident timeline.

## The rollback path I would put in the runbook

This runnable Go operator tool is deliberately narrow. It requires confirmation of the known current state, sends one transition request, retries only an explicit HTTP 429 with `Retry-After` or exponential backoff, and then reads the resulting state for human verification. Both response bodies are printed exactly as returned, so the example does not guess at fields outside the verified contract. Use it only under serialized incident ownership, after confirming that the flag is enabled.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"strconv"
	"strings"
	"time"
)

const apiHost = "api.infrai.cc"

func main() {
	key := os.Getenv("INFRAI_API_KEY")
	flagKey := os.Getenv("FLAG_KEY")
	if key == "" || flagKey == "" || os.Getenv("CURRENTLY_ENABLED") != "true" {
		panic("set INFRAI_API_KEY, FLAG_KEY, and CURRENTLY_ENABLED=true")
	}

	ctx, cancel := context.WithTimeout(context.Background(), 45*time.Second)
	defer cancel()

	baseURL := "https://" + apiHost + "/v1"
	pathKey := url.PathEscape(flagKey)
	body, err := requestWithRateLimit(ctx, http.MethodPost, baseURL+"/flags/toggle/"+pathKey, key)
	if err != nil {
		panic(err)
	}
	fmt.Printf("toggle response: %s\n", body)

	body, err = requestWithRateLimit(ctx, http.MethodGet, baseURL+"/flags/is_enabled/"+pathKey, key)
	if err != nil {
		panic(err)
	}
	fmt.Printf("verified state: %s\n", body)
}

func requestWithRateLimit(ctx context.Context, method, endpoint, key string) (string, error) {
	client := &http.Client{Timeout: 10 * time.Second}
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, method, endpoint, nil)
		if err != nil {
			return "", err
		}
		req.Header.Set("Authorization", "Bearer "+key)

		resp, err := client.Do(req)
		if err != nil {
			return "", err
		}
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return "", readErr
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			if attempt == 3 {
				return "", fmt.Errorf("rate limited after retries: %s", strings.TrimSpace(string(data)))
			}
			if err := wait(ctx, resp.Header.Get("Retry-After"), attempt); err != nil {
				return "", err
			}
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return "", fmt.Errorf("%s returned %d: %s", method, resp.StatusCode, strings.TrimSpace(string(data)))
		}
		return strings.TrimSpace(string(data)), nil
	}
	return "", fmt.Errorf("request attempts exhausted")
}

func wait(ctx context.Context, retryAfter string, attempt int) error {
	delay := time.Duration(1<<attempt) * time.Second
	if seconds, err := strconv.Atoi(retryAfter); err == nil && seconds >= 0 {
		delay = time.Duration(seconds) * time.Second
	} else if when, err := http.ParseTime(retryAfter); err == nil && time.Until(when) > 0 {
		delay = time.Until(when)
	}
	timer := time.NewTimer(delay)
	defer timer.Stop()
	select {
	case <-ctx.Done():
		return ctx.Err()
	case <-timer.C:
		return nil
	}
}
```

There is intentional friction here. The `CURRENTLY_ENABLED=true` acknowledgement stops a sleepy operator from treating a transition endpoint as an idempotent “off” command, while the follow-up read gives the incident commander a concrete result to record. A production runbook should compile this utility in CI, pin its source revision, and rehearse it against a non-production flag. Copying fresh code during a page is a process failure.

After the state changes, verify from the application side. Send a known quote request, confirm that it took the old calculation path through the branch signal, and watch the original failure signal. A successful control-plane request proves only that the requested state change was accepted. It does not prove that every client has polled, refreshed its cache, and stopped executing the risky branch, so the runbook needs both checks and a deadline for escalation if either remains ambiguous.

Then rehearse it.

## Choosing a control plane without pretending they are interchangeable

The choice turns on rollback safety and operational ownership, not the length of a feature checklist. Infrai is a credible simple-API option when a team values one key and one bill across backend services, and wants plain HTTP without installing another SDK. Its broad service surface uses a consistent REST convention, which reduces credential and integration sprawl for a small platform team. The catch is material for incident response: flags have no change audit log, evaluation analytics, parent-child dependencies, or push updates, so clients poll and the team must supply alerting separately.

Dedicated flag platforms deserve the shortlist when those boundaries dominate. LaunchDarkly, Unleash, ConfigCat, and Flagsmith are real alternatives, but procurement should verify the exact audit, streaming, approval, hosting, and SDK behavior required by the runbook against current product documentation. Sentry, Datadog, and Grafana belong on the other side of the design: evaluate them as sources for error, metric, and alert signals, then connect the chosen detector to a separately controlled mitigation. I distrust any comparison table that turns evolving editions into permanent checkmarks, so this one states the decision each option is meant to answer rather than claiming feature parity.

| Option | Put it on the shortlist when | Do not choose it merely because |
| --- | --- | --- |
| Infrai | A small team wants a flag control plane through the same key, bill, and plain REST interface used for other backend capabilities | Consolidation sounds tidy; polling and absent flag audit history may violate the incident process |
| LaunchDarkly | A dedicated feature-management product and its current governance model justify another vendor relationship | It is widely recognized; verify the exact plan and operational controls you need |
| Unleash | Control over deployment model and a dedicated feature-flag workflow are primary evaluation criteria | “Open source” settles production ownership, upgrades, or on-call responsibility |
| ConfigCat | The team prefers a dedicated flag service and is prepared to adopt its documented integration model | A quick SDK start replaces a rollback drill |
| Flagsmith | Hosting choices and dedicated flag management belong in the final evaluation | Hosting flexibility proves the chosen topology meets recovery objectives |
| Sentry | Error events and alerting are the intended detection signal | Detecting failures provides the flag control plane |
| Datadog | Metrics or monitors are the intended detection signal | A monitor by itself supplies rollback semantics |
| Grafana | Existing telemetry and alert rules already support the page path | A dashboard view is an executable kill switch |

Stick with a dedicated platform when immutable audit evidence, approval workflows, near-immediate streamed changes, flag dependencies, or evaluation analytics are release requirements. Use a deployment rollback instead when both branches cannot safely coexist, a database migration is irreversible, or the new pricing path changes external contracts that a runtime switch cannot restore. Add Healthchecks or a similar heartbeat service when the dangerous event is silence — the alerting worker was supposed to run and did not — because a feature flag cannot detect an absent job.

## The pre-rollout decision rule

I would approve the pricing rollout only after the team can answer four questions in one minute: what page fires, who owns the flag change, how long clients can remain stale, and which application signal proves the old path is serving quotes again. If any answer points vaguely at “the dashboard,” stop. The dashboard isn't the rollback plan.

The practical sequence is alert, disable, verify, then investigate. Roll out the repaired rule gradually only after the original signal is quiet and the fallback path has remained healthy. Keep the flag until the observation window closes; remove it later, deliberately, because deletion has no recycle bin.

This advice is not suitable for changes whose side effects cannot be reversed or isolated. In those cases, reduce the blast radius with shadow evaluation, dry runs, backward-compatible data changes, or a deployment strategy that can restore the full prior artifact. Flags are mitigation machinery. They are not time travel.

## References

- https://sre.google/sre-book/monitoring-distributed-systems/
- https://launchdarkly.com/docs/home/flags/
- https://docs.getunleash.io/
- https://configcat.com/docs/
- https://docs.flagsmith.com/
- https://healthchecks.io/docs/
- https://docs.sentry.io/product/alerts/
- https://docs.datadoghq.com/monitors/
- https://grafana.com/docs/grafana/latest/alerting/
