# Backend Custom Metrics for Pricing Analytics (When Feature Flags Lie)

The product analytics dashboard says `pricing_rule_mismatch_rate > 1%`, but that isn't yet an actionable incident. Backend custom metrics must show which rule revision was evaluated, which tenant segment received it, whether invoices or previews were affected, and whether disabling the feature flag will stop new bad decisions without corrupting historical reporting. Flag stats cannot answer all of that. An analytics API alone cannot prove which rollout path produced the result either.

TL;DR: use backend custom metrics as the record of pricing outcomes and feature-flag statistics as rollout evidence. Join them with a bounded, low-cardinality release key such as `rule_revision`; keep tenant identifiers out of metric labels; and make the page fire on a customer-impact ratio that names the rollback unit. The flag answers "who was exposed?" The backend answers "what happened?" Roll back from the intersection, not from either dashboard in isolation.

This distinction matters for a new B2B SaaS pricing rule because the safest reversal is rarely "turn everything off." A single enterprise contract may have an override, a preview request may never become an invoice, and retries can inflate event counts. The useful signal is narrower: among finalized pricing decisions for the candidate revision, how many violated an invariant that the old rule satisfied?

## Should a product analytics dashboard use backend metrics or flag stats?

A page should identify an action, not announce that a chart moved. For this rollout, a useful notification is: "Candidate pricing revision has 37 mismatches across 2,941 finalized decisions in the last 10 minutes; disable revision `r17` for the affected segment." Those numbers are an example alert payload, not a benchmark or a recommended universal threshold. The alert also needs a link to the runbook and the deployment or configuration change that can reverse exposure.

The numerator must describe a real invariant failure: a negative payable amount, a currency mismatch, a missing contract override, or a material disagreement between the candidate and shadow calculation. The denominator must count eligible finalized decisions, not requests, flag checks, page views, or retries. Otherwise a traffic spike changes the ratio without changing customer harm.

One question comes before every dashboard review: **what page fired?** If the answer is "flag evaluation volume changed," there is still no demonstrated pricing failure. If the answer is "pricing mismatches increased," but the event lacks the evaluated rule revision, the responder still cannot choose a safe rollback target. The page must carry both facts, including when a Node.js service supplies the custom metric through an API.

## Work backward from impact to exposure

Start with the final pricing decision and trace backward. The backend knows the contract inputs, the chosen rule revision, the computed amount, whether the operation was a preview or finalization, and whether an invariant failed. The flag system knows the targeting decision and rollout state. Neither source should impersonate the other.

| Evidence | Backend custom metric | Flag statistic | Operational use |
|---|---:|---:|---|
| Finalized pricing decisions | Yes | No | Denominator for impact |
| Invariant failures | Yes | No | Numerator for the page |
| Candidate revision selected | Yes | Yes | Correlation key |
| Targeted population exposed | Partial | Yes | Rollout verification |
| Contract-specific reason | Logs or traces, not labels | No | Investigation |
| Safe rollback unit | Derived from both | Derived from both | Response decision |

That last row is the point. Exposure without outcome is speculation; outcome without exposure is slow forensics. The join does not require a customer identifier in every time series. Putting `tenant_id`, contract IDs, invoice IDs, or raw error text into labels makes the metric surface harder to operate and expands the places where customer-linked data may persist. GDPR Article 17 establishes a right to erasure under specified conditions, so identifiers that need individual deletion belong in a controlled event store with retention and deletion procedures, not casually copied into aggregate metric dimensions.

Use a small set of dimensions that correspond to actions: `rule_revision`, `decision_kind`, `result`, and perhaps a coarse rollout segment whose membership is governed elsewhere. Detailed diagnosis belongs in correlated logs or traces with access controls. Keep the metric for counting.

## Instrument the decision boundary

Instrumentation should happen after the application has resolved the flag and calculated the price, at the boundary where it can distinguish a preview from a committed decision. The counter below uses a generic interface so the operational contract is visible without binding the service to a particular metrics backend.

```go
package pricing

import (
    "context"
    "errors"
)

type Counter interface {
    Add(ctx context.Context, value int64, labels map[string]string)
}

type Decision struct {
    RuleRevision string
    Kind         string // "preview" or "final"
    Currency     string
    AmountMinor  int64
}

type Recorder struct {
    decisions  Counter
    mismatches Counter
}

func (r Recorder) Record(ctx context.Context, d Decision, expectedCurrency string) error {
    if d.RuleRevision == "" {
        return errors.New("missing pricing rule revision")
    }

    labels := map[string]string{
        "rule_revision": d.RuleRevision,
        "decision_kind": d.Kind,
    }
    r.decisions.Add(ctx, 1, labels)

    result := "ok"
    if d.AmountMinor < 0 {
        result = "negative_amount"
    } else if d.Currency != expectedCurrency {
        result = "currency_mismatch"
    }
    if result != "ok" {
        r.mismatches.Add(ctx, 1, map[string]string{
            "rule_revision": d.RuleRevision,
            "decision_kind": d.Kind,
            "result":        result,
        })
    }
    return nil
}
```

The deliberate omission is more important than some fields: there is no `tenant_id`. There is also no price amount as a label, no free-form exception message, and no flag name that can vary per customer. The example records one decision per call; production code must place that call where retries cannot count the same finalized operation twice, or use a stable event pipeline that deduplicates before aggregation.

The first instrumentation attempt often counts HTTP requests because requests are convenient. That fails under retries, batch operations, and preview traffic. Count the domain decision instead.

Requests lie.

So do percentages without denominators.

A shadow calculation can detect disagreement before exposure grows. Run the current and candidate rules against the same eligible input, compare invariant-relevant outputs, and emit the candidate revision with the comparison result. Do not put both full price objects into metrics. Store diagnostic detail in the controlled record used for investigation, with an explicit retention policy.

## Decide rollback safety before rollout

A reversible flag does not make an irreversible side effect reversible. Before enabling the rule, define the boundary: disabling `r17` stops future evaluations, while already finalized invoices, sent notifications, ledger writes, or exported records follow their own correction procedure. The runbook should say which of those happened before the page could fire.

Use this decision rule without pretending one threshold fits every business:

1. Page only on finalized customer-impact signals, segmented by the candidate revision.
2. Treat preview and shadow mismatches as rollout gates or tickets unless they predict an imminent committed failure.
3. Disable the smallest affected rollout segment when the invariant breach is attributable to that segment and revision.
4. Stop the entire candidate revision when attribution is incomplete, because uncertain targeting is a poor foundation for a partial rollback.
5. Reconcile committed side effects separately; never describe flag disablement as data repair.

This is a trade-off, and the reason for choosing the coarse key is operational rather than aesthetic. Coarse segments make rollback comprehensible but reduce targeting precision. Fine segments can limit blast radius, yet they multiply time series and make an exhausted responder compare tiny populations with unstable ratios. A coarse, well-defined rollback key in the page is preferable; retrieve tenant-level evidence from a protected event record after containment. The cost is slower tenant-by-tenant diagnosis after the immediate rollback. The benefit is that the person holding the pager doesn't have to infer a containment boundary from hundreds of sparse series while pricing decisions continue.

Test the telemetry as part of the release. Feed known preview and final decisions through both rule revisions; verify the denominator increments once, force each invariant failure, and confirm the alert groups by the exact key the flag operator can disable. Then rehearse disabling the candidate while requests are in flight. A dashboard screenshot is not evidence that this path works.

## The earlier signal and its false-positive bill

The page is late by design: it waits for finalized impact. The signal that should fire earlier is the candidate-versus-current mismatch ratio from shadow or preview evaluations, gated by a minimum sample count and separated by decision kind. It can halt expansion before committed harm, but it should not wake someone merely because five unusual previews disagree.

Thresholds have two dimensions, not one: error ratio and evidence volume. A 1% mismatch ratio after 10,000 eligible decisions means something very different from one mismatch after three decisions, even though the latter displays as 33%. The exact limits must come from the pricing invariant, traffic distribution, and acceptable exposure window; none can be inferred from a generic observability recipe.

False positives carry a concrete operational cost. They interrupt the responder, teach the team to distrust the dashboard, and can trigger a rollback that denies customers the intended rule even when no finalized decision was wrong. False negatives carry the opposite cost: more bad decisions before containment. Keep the customer-impact page conservative and actionable, then route the noisier early-warning signal to automated rollout pause or staffed review when the organization can support it.

The durable design is two linked controls: an early mismatch gate for rollout progression and a finalized-impact page for incident response. Both share `rule_revision`; neither treats flag evaluation counts as product outcomes. When the alert arrives at 3 a.m., the responder should be able to name the failing invariant, the exposed revision, the rollback unit, and the side effects that remain. Anything less is a graph asking a human to invent a diagnosis.

## Further reading

- GDPR Article 17, Right to erasure: https://gdpr-info.eu/art-17-gdpr/
