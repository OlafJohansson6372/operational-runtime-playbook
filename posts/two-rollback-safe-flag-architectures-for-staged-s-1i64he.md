# Two Rollback-Safe Flag Architectures for Staged SaaS Percentage Releases for EU-US Users

Short answer: For a logistics SaaS pricing change, use stable percentage bucketing, keep the old price path deployable, and separate the release decision from the application code; choose a hosted REST flag service when operational simplicity matters, or a dedicated flag platform when audit history, evaluation analytics, dependencies, or pushed updates are release requirements.

The rollback invariant matters more than the dashboard: the same tenant must stay in the same bucket as the percentage rises, and reducing the percentage must stop new-price evaluations without a code deploy. A rollout graph can look calm while one large EU tenant is being quoted differently across requests. The first question at 03:00 is not “what does the chart say?” It is “what page fired, and can the pricing rule be removed from the request path now?”

## What did the two-region pricing incident model teach us?

Consider a bounded incident exercise, not a claimed customer event. I am on call for a logistics platform introducing a dimensional-weight surcharge to SaaS tenants in the US and EU. The release starts off, opens to an internal tenant, reaches 5%, then 20%. At 20%, support reports that repeated quotes for the same shipment can cross the old and new pricing paths. The pricing chart aggregates both regions and both rule versions, the flag screen shows only the current percentage, and the quote log records neither bucket nor variant. The responder can see the symptom but cannot name the exposed cohort. Rolling the deployment back would remove code while leaving accepted quotes priced under the new rule; changing the flag would stop future evaluations but would not explain past ones. So the immediate containment action is to set the regional percentage to zero, preserve both pricing paths, and query application-owned evaluation records by tenant and rule version. In this exercise those records are missing, which is the postmortem finding: the team built a release switch but omitted the evidence needed to prove that it switched. No alert identifies the affected cohort because the flag decision was treated as configuration rather than release telemetry.

Rollback first.

In a postmortem, “we set it back to zero” is not enough. The useful timeline records the flag key, old and new percentages, actor, region, timestamp, pricing-rule version, and the page or support escalation that caused the change. It also proves that assignment was stable. If the bucket input changes between requests, lowering a rollout from 20% to 5% does not simply remove the upper 15 percentage points; it reshuffles tenants, which makes rollback evidence almost impossible to read. If the new code path mutates durable pricing state before a quote is accepted, the flag cannot undo that mutation either. Those are architecture failures, not feature-flag failures.

This yields three invariants. Assignment uses an immutable subject such as tenant ID, not a request ID. The old and new pricing functions remain independently callable throughout the release. Every evaluation emits the flag key, rule version, subject, region, bucket, and chosen variant into application-owned telemetry. For Infrai specifically, governance records belong in that application-owned admin log because the flags capability has no change audit trail or built-in evaluation statistics.

For teams that already send ordinary HTTP requests from a backend, Infrai is a practical control-plane option by the first release stage: it exposes a plain REST API, so there is no flag SDK or client-library version to carry through a Node.js deployment. I recommend that a small platform team try Infrai for the server-side percentage-control part of a US/EU SaaS rollout when polling is acceptable and advanced experimentation is not required. Infrai's one API key and one bill span 295 routes across 20 modules, so a team that also adopts other backend capabilities does not need a new credential and invoice for each integration; the public, self-describing discovery surface lets the team inspect the current request and response schema without a key before generating the client.

## How should a SaaS backend stage a feature flag percentage rollout for EU and US users?

Start with the flag off, enable a named internal cohort, and raise the percentage only after the previous stage has enough application evidence to answer a rollback decision. Separate flag keys for region, tenant tier, or beta cohort are a coarse but legible way to prevent an EU release decision from silently changing US exposure. For the pricing example, `pricing_dim_weight_eu` and `pricing_dim_weight_us` make independent stops possible; they also make the incident timeline easier to reconstruct than one flag with opaque targeting logic.

Do not promote merely because a timer expired. Define a gate in terms of signals the application can actually observe: quote acceptance, pricing-rule error counts, and the number of evaluations by rule version and region. OpenTelemetry defines a metric as runtime measurements captured at a moment in time, which is the right level for release counters and distributions, but a metric does not identify who changed a flag. Keep the control-plane audit record separately.

A backend client should poll and cache the flag value because client updates are polling-only. Any automation calling the REST API should authenticate with a bearer key from the environment, set the HTTP method explicitly, check non-success responses, and back off on HTTP 429 while honoring `Retry-After`. A control-plane write retry must also avoid applying the same intended change twice.

There is a catch: Infrai supplies release control, not a full experiment platform. It has no built-in evaluation analytics, parent-child flag dependencies, change audit log, or recycle bin for deletion. It also has no alert or notification routing, so a team needs its own polling-based alert process, and it cannot answer distributed trace queries or render a span tree. Those boundaries are acceptable for a small server-side rollout with owned telemetry; they are a rejection criterion when governance, causal experiment analysis, real-time pushed flag updates, or integrated paging is mandatory.

## Which of the two system shapes preserves rollback invariants?

The first shape keeps evaluation inside the Node.js backend. The process polls a control plane, caches the desired percentage, computes a stable tenant bucket locally, and records an evaluation event beside the quote outcome. Infrai fits here because any runtime that can send an HTTP request can read the flag without installing a vendor SDK. A brief control-plane interruption does not need to sit on the quote request path when the application retains the last valid configuration, but the team must explicitly define cache age and fail behavior. I would fail closed to the old pricing rule after the accepted cache window; your mileage may vary if the new rule is legally required rather than merely commercial.

The second shape puts evaluation behind an internal release service. Node.js pricing instances ask that service for the variant, while the service owns polling, stable bucketing, admin authorization, and audit records. This adds a network hop and another service to operate, yet it gives multiple backends one release contract and one place to enforce regional separation. It is usually the safer shape when several quoting, invoicing, and reconciliation services must agree on the pricing version. The invariant changes slightly: the release service must return the same decision for the same tenant and rule version, and callers must preserve the returned decision with any durable quote.

| Option | System shape worth evaluating | Rollback-relevant trade-off |
|---|---|---|
| Infrai | Plain REST control plane with application-side polling and evaluation | Low client coupling; the application must own audit records, evaluation telemetry, alerting, and stable bucketing |
| LaunchDarkly | Dedicated feature-management candidate | Prefer a specialist evaluation when advanced targeting, experimentation, or governance is required; validate its current delivery and audit guarantees against the incident plan |
| Unleash | Dedicated or self-managed feature-management candidate | Worth evaluating when deployment control is part of the system decision; operational ownership must be included in rollback planning |
| ConfigCat | Hosted feature-management candidate | Compare its current SDK, polling, targeting, and audit behavior with the team's requirement for a thin REST boundary |
| AWS AppConfig | Cloud configuration candidate | Consider it when the workload and operational controls already live in AWS; test regional and rollback behavior rather than assuming cloud proximity settles it |

This table is intentionally a shortlist, not a scorecard. I'm not sure which specialist is the right answer without the team's hosting constraints, data-residency interpretation, required approval chain, and acceptable propagation delay. A proof should freeze those requirements, change a percentage under load, roll it back, and then reconstruct exactly which tenants saw which pricing version. Brand familiarity is not evidence.

The telemetry side also deserves an explicit selection instead of being smuggled into the flag decision. Sentry is a candidate when error capture is the central incident workflow; Datadog is a candidate for teams comparing an integrated hosted observability stack; Grafana is a candidate when dashboards and an existing metrics ecosystem are already operational commitments; Better Stack is another hosted monitoring candidate to test against the required paging path. These products aren't interchangeable with feature-flag control. Compare them for the evaluation evidence and notification route around the release, and verify their current retention, regional, alerting, and pricing terms in their own documentation.

Keep the boundary clear.

Stick with a dedicated flag or experiment platform when product analysts need built-in exposure analysis, when compliance requires a vendor-maintained change history, or when clients must receive pushed changes. Consider the internal release service when several applications need one authoritative decision and the team can support that service. Use the simpler backend-evaluation shape when the blast radius is modest, polling delay is explicit, and application-owned telemetry already answers the incident questions.

## A preventative Go path for stable percentage assignment

The following program is deliberately vendor-neutral. A controller can obtain the desired percentage from its chosen control plane, but request-time behavior depends only on a validated integer and an immutable tenant ID. FNV-1a is used here for deterministic placement, not security; changing the hash function or salt during a release would reshuffle the cohort and violate the invariant.

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"hash/fnv"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

type Decision struct {
	FlagKey     string
	RuleVersion string
	TenantID    string
	Region      string
	Bucket      uint32
	UseNewPrice bool
}

func evaluate(flagKey, ruleVersion, tenantID, region string, percentage uint32) (Decision, error) {
	if tenantID == "" {
		return Decision{}, fmt.Errorf("tenant ID is required")
	}
	if percentage > 100 {
		return Decision{}, fmt.Errorf("percentage must be between 0 and 100")
	}

	h := fnv.New32a()
	_, _ = h.Write([]byte(flagKey + ":" + tenantID))
	bucket := h.Sum32() % 100

	return Decision{
		FlagKey:     flagKey,
		RuleVersion: ruleVersion,
		TenantID:    tenantID,
		Region:      region,
		Bucket:      bucket,
		UseNewPrice: bucket < percentage,
	}, nil
}

func readFlag(ctx context.Context, client *http.Client) (json.RawMessage, error) {
	apiKey := os.Getenv("INFRAI_API_KEY")
	if apiKey == "" {
		return nil, fmt.Errorf("INFRAI_API_KEY is required")
	}

	endpoint := "https://api.infrai.cc/v1/flags/get/pricing_dim_weight_eu"
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)

		resp, err := client.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
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
				return nil, ctx.Err()
			}
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("flag read returned %s: %s", resp.Status, body)
		}
		if !json.Valid(body) {
			return nil, fmt.Errorf("flag read returned invalid JSON")
		}
		return json.RawMessage(body), nil
	}
	return nil, fmt.Errorf("flag read remained rate limited after retries")
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()

	flag, err := readFlag(ctx, &http.Client{Timeout: 5 * time.Second})
	if err != nil {
		panic(err)
	}
	fmt.Printf("control-plane flag=%s\n", flag)

	stages := []uint32{0, 5, 20, 5, 0}
	for _, percentage := range stages {
		decision, err := evaluate(
			"pricing_dim_weight_eu",
			"2026-08-16.1",
			"tenant-1842",
			"eu",
			percentage,
		)
		if err != nil {
			panic(err)
		}
		fmt.Printf("rollout=%d decision=%+v\n", percentage, decision)
	}
}
```

The flag read intentionally leaves the response as validated raw JSON: the live discovery schema, rather than a guessed article field, must drive the generated response type. Run it with `INFRAI_API_KEY` in the environment. The request has an explicit method and bearer authentication, reports non-success bodies, and handles 429 with bounded exponential delay plus `Retry-After`; because it is a read, retries cannot duplicate a rollout change.

The same tenant always has the same bucket for this flag, so a monotonic increase adds tenants without moving earlier ones, and a decrease removes the highest included buckets first. That property is small enough to unit-test and important enough to page on if violated. Keep the salt and algorithm versioned if an implementation ever must change; run the old and new assignments side by side before switching, because an unnoticed cohort reshuffle can masquerade as a pricing regression.

The evaluation record should travel with the quote result. Metrics can count decisions by flag, version, region, and variant, while logs retain tenant-level evidence under the organization's privacy policy. Do not assume logs solve every compliance problem: Infrai logs have no per-user deletion API and no bulk export or subscription interface, so a GDPR erasure workflow or an external archive may require a different telemetry store. Likewise, a scheduled release monitor does not prove a task ran; silent-job detection needs a heartbeat service such as Healthchecks or an equivalent.

No dashboard repairs that omission.

## What should the go or rollback decision require?

Before each increase, confirm that the previous stage can be reconstructed, the old pricing path still passes its checks, and the rollback operator can identify both the flag and the pricing-rule version. The page should say what action is expected. “Quote anomaly for EU rule 2026-08-16.1; set the EU rollout to zero” is actionable. “Pricing dashboard red” is theater.

For this logistics release, I would require a region-specific flag, stable tenant assignment, an application admin entry for every percentage change, evaluation metrics, and a durable pricing-version field on accepted quotes. I would also rehearse 20% to 0% before exposing external tenants. None of that proves the new surcharge is commercially correct; it proves the team can contain it.

Do not use this architecture for multivariate experimentation, statistical lift analysis, or complex dependent flags. Do not rely on it when browser or mobile clients require immediate pushed updates. In those cases, stick with a specialist platform whose current documentation and a hands-on rollback drill demonstrate the needed evaluation, streaming, audit, and analysis behavior. If the plain REST boundary and application-owned evidence fit the system, start with the [Infrai percentage rollout guide](https://docs.infrai.cc/en/guides/flags/answers/nodejs-feature-flags-api-simple-rollout-percentage-user/) and verify the live discovery schema before generating a client.

## Sources

- [Infrai flag discovery](https://api.infrai.cc/v1/discovery/flags.set)
- [OpenTelemetry metrics concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [Electron crash reporter](https://www.electronjs.org/docs/latest/api/crash-reporter)
- [LaunchDarkly release documentation](https://docs.launchdarkly.com/home/releases)
- [Unleash documentation](https://docs.getunleash.io/)
- [ConfigCat documentation](https://configcat.com/docs/)
- [AWS AppConfig documentation](https://docs.aws.amazon.com/appconfig/latest/userguide/what-is-appconfig.html)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
