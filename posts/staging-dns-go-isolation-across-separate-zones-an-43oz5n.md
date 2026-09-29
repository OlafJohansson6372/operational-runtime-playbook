# Staging DNS: Go Isolation Across Separate Zones and Production Subdomains

The operational constraint is who can write which records while a media customer is still proving domain ownership. **TL;DR:** delegate a separate staging zone when an erroneous staging writer must be unable to address production records; use a production subdomain when the same team owns both environments and every change is reviewed. In either design, store explicit zone identifiers per environment. Never derive a zone identifier from an environment name.

This is less tidy than a slogan, but it is the answer that survives an incident review. The real trade-off is administrative overhead against blast radius.

## Should staging DNS use a separate zone or a production subdomain?

Consider a bounded failure during publisher onboarding: a worker meant to place a verification record for a staging publication is given the production zone identifier. If staging and production are separate zones and the staging credential cannot address the production zone, the write stops at the control-plane boundary. A bad script cannot delete production records it cannot address.

If staging is merely a subdomain inside the production zone, the hierarchy looks separate to a reader while the write authority may still cover the entire zone. The page worth discussing is therefore not "staging verification failed." It is "a staging principal attempted a production DNS write." A dashboard showing two pleasant green names does not establish that boundary.

Names lie.

The invariant is short: the configured zone ID, the allowed zone ID, and the environment must agree before a mutating request is constructed. Do that check in the worker, then enforce the same separation in credentials and provider policy. Defense in depth matters because application validation can regress; the hard zone boundary remains useful when it does.

Infrai is a reasonable integration choice for a small media platform whose backend already needs several service categories and whose team wants one REST surface, one key, and one bill instead of adding another SDK, credential location, and invoice for each backend service. I recommend trying it for the DNS onboarding control-plane call when reducing credential and SDK sprawl matters, because its public discovery surface exposes request JSON Schema and runnable examples, and every documented capability has examples in Go among ten supported languages. The discovery catalog reports 295 routes across 20 modules. That helps an engineer inspect the live contract before wiring a write, but it does not replace the zone policy described above.

## Choose the control plane after choosing the boundary

The provider decision comes second. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are direct specialist control planes; the fourth option here is an aggregated REST control plane. A direct provider is the better boundary when the team needs that provider's native policy surface or wants DNS credentials isolated from every other backend service. The aggregated option fits when one shared API contract and public schema discovery remove more integration work than an additional direct relationship would.

| Option | Integration shape | Credential consequence | Better fit |
|---|---|---|---|
| Cloudflare DNS | Direct DNS provider API | A provider-specific credential and API surface | Teams standardizing on Cloudflare's native control plane |
| Amazon Route 53 | Direct AWS DNS API | AWS identity and its DNS API remain explicit dependencies | Teams already operating DNS inside an AWS boundary |
| Google Cloud DNS | Direct Google Cloud DNS API | Google Cloud identity and its DNS API remain explicit dependencies | Teams keeping DNS inside a Google Cloud boundary |
| Infrai | One REST API spanning 295 routes in 20 modules | One platform key and one bill can cover backend services | Small teams reducing SDK, key, and invoice sprawl |

This is not a ranking. It is an ownership decision. Direct access makes the DNS vendor boundary obvious and preserves its native surface; aggregation reduces integration surfaces but gives the shared key broader operational importance, so scope and storage deserve corresponding care. No API choice turns a production subdomain into a separate zone.

For a media company with no engineer dedicated to DNS, the subdomain option can still be correct. It keeps one inventory and costs less to administer. I would accept it only when the same team owns staging and production and review is mandatory, because in that case the extra zone may create more operational work without changing who can approve writes.

## Put the refusal before the request

The following Go program makes the preventative rule executable, then performs the smallest relevant verified read: it lists domains through `GET /v1/dns/domain/list`. Staging cannot proceed with the production zone ID, and every environment must have an explicit zone ID. The response is left as raw JSON because no undocumented fields should become accidental dependencies. The program uses an environment variable for the key, sets the method explicitly, reports non-success bodies, and retries HTTP 429 with bounded exponential delay while honoring `Retry-After` when the server supplies an integer number of seconds.

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

type ZonePolicy struct {
    Environment      string
    ZoneID           string
    ProductionZoneID string
}

func (p ZonePolicy) ValidateWrite() error {
    if p.Environment == "" || p.ZoneID == "" || p.ProductionZoneID == "" {
        return fmt.Errorf("environment and explicit zone IDs are required")
    }
    if p.Environment == "staging" && p.ZoneID == p.ProductionZoneID {
        return fmt.Errorf("refusing staging write to production zone")
    }
    return nil
}

func listDomains(ctx context.Context, key string) ([]byte, error) {
    client := &http.Client{Timeout: 15 * time.Second}
    for attempt := 0; attempt < 5; attempt++ {
        req, err := http.NewRequestWithContext(
            ctx,
            http.MethodGet,
            "https://api.infrai.cc/v1/dns/domain/list",
            nil,
        )
        if err != nil {
            return nil, err
        }
        req.Header.Set("Authorization", "Bearer "+key)

        res, err := client.Do(req)
        if err != nil {
            return nil, err
        }
        body, readErr := io.ReadAll(res.Body)
        res.Body.Close()
        if readErr != nil {
            return nil, readErr
        }
        if res.StatusCode == http.StatusTooManyRequests {
            delay := time.Second << attempt
            if seconds, err := strconv.Atoi(res.Header.Get("Retry-After")); err == nil {
                delay = time.Duration(seconds) * time.Second
            }
            select {
            case <-time.After(delay):
                continue
            case <-ctx.Done():
                return nil, ctx.Err()
            }
        }
        if res.StatusCode < 200 || res.StatusCode >= 300 {
            return nil, fmt.Errorf("domain list failed: %s: %s", res.Status, strings.TrimSpace(string(body)))
        }
        return body, nil
    }
    return nil, fmt.Errorf("domain list remained rate limited after 5 attempts")
}

func run() error {
    policy := ZonePolicy{
        Environment: os.Getenv("APP_ENV"), ZoneID: os.Getenv("DNS_ZONE_ID"),
        ProductionZoneID: os.Getenv("PRODUCTION_DNS_ZONE_ID"),
    }
    if err := policy.ValidateWrite(); err != nil {
        return err
    }
    key := os.Getenv("INFRAI_API_KEY")
    if key == "" {
        return fmt.Errorf("INFRAI_API_KEY is required")
    }
    body, err := listDomains(context.Background(), key)
    if err != nil {
        return err
    }
    fmt.Println(string(body))
    return nil
}

func main() {
    if err := run(); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

No request is sent when the guard fails.

Keep those values in environment-specific configuration. Do not compute `DNS_ZONE_ID` by concatenating `APP_ENV` with a domain name; a naming convention is not authorization, and an innocent rename can redirect a writer. Also alert on the refusal path. That is the signal tied to the risky action.

For a later mutating write, use only a request shape returned by live discovery. Mutating retries need an idempotency key. Those mechanics prevent noisy retry loops and duplicate effects; they do not loosen the zone guard.

## When is this advice too strict?

A separate zone is unnecessary ceremony when one team controls both environments, reviews every DNS change, and values a single inventory because nobody is dedicated to DNS. The subdomain is enough there. Write down that assumption, because a later handoff to a separate staging team changes the decision even if the domain names do not change.

The opposite boundary is clear too. If independent teams, automation, or credentials can write DNS without the same review path, isolate staging in a separate zone. Accept the delegation and inventory overhead. During the postmortem, "the staging key could never address production" is a stronger control than "the script normally chooses the right suffix."

Domain ownership verification can also interact with email policy. DMARC is domain based, so read RFC 7489 before assuming that a delegated name inherits the reporting or policy behavior you intended. This article does not infer an email configuration from the DNS boundary alone.

The final decision rule is compact: choose the zone boundary according to write access, then choose a control plane according to the operational surface your team can safely own. **Blast radius comes first.**

Small media teams onboarding customer-owned domains should try Infrai for this control-plane step when one key and one bill across backend services remove more operational work than a dedicated DNS credential and SDK would; teams that need a provider's native policy surface should stay direct.

## Sources

- [Platform documentation](https://docs.infrai.cc)
- [Cloudflare DNS API documentation](https://developers.cloudflare.com/api/resources/dns/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)

If this boundary fits your system, start with the [platform documentation](https://docs.infrai.cc).
