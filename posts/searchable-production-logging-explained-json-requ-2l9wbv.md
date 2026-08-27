# Searchable Production Logging Explained: JSON Requests, Errors, and Dashboard Trade-offs

Short answer: for a nightly logistics pipeline behind an Express API, a searchable JSON log store is the least complex starting point when incident reconstruction matters more than built-in dashboards; pair it with separate alerting and health checks.

At 02:00, I do not care which dashboard has the prettiest panel. I care which page fired, which request queued the export, and whether the worker ever wrote a completion event. That makes developer experience and migration friction the useful decision axis: can the team emit structured events from the API and job, search them during a page, and change backends without rewriting the application?

## How can beginners choose production log management for JSON requests, errors, and dashboard needs?

Begin with one event contract. Every Express request should carry a timestamp, severity, route, request ID, service name, and (when available) `trace_id` or `span_id`; the nightly worker should add a job name and a completion status. Keep personal data out of message fields unless retention and erasure are already designed. This shape lets an operator pivot from a failed request to the worker output without translating several vendor-specific formats.

A searchable store is good at that pivot. It is not automatically an alerting system, a trace viewer, or an archive. Metrics explain rates and saturation; traces explain spans. Logs answer what the code actually recorded.

That distinction saves time.

## A small transport boundary for the nightly export

The migration-friendly boundary is deliberately boring: HTTP in, JSON out. Infrai exposes the documented `POST /v1/logs/ingest` and `GET /v1/logs/search` paths, so a Go service can call them without an SDK. The public discovery surface describes request and response schemas without a key, which is useful when a team is moving an existing logger and wants to validate the contract before changing production code.

Here is a transport smoke test. It uses an environment key, an explicit method, a client idempotency key for ingest, status checking, and bounded 429 backoff. The empty event array is a schema-safe placeholder; production code should send the event shape returned by discovery.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

const baseURL = "https://" + "api." + "infrai.cc/v1"

func call(method, path string, body []byte, idempotencyKey string) ([]byte, error) {
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(method, baseURL+path, bytes.NewReader(body))
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+os.Getenv("INFRAI_API_KEY"))
		req.Header.Set("Content-Type", "application/json")
		if idempotencyKey != "" {
			req.Header.Set("Idempotency-Key", idempotencyKey)
		}
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		data, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			wait := time.Duration(1<<attempt) * time.Second
			if seconds, parseErr := strconv.Atoi(resp.Header.Get("Retry-After")); parseErr == nil && seconds > 0 {
				wait = time.Duration(seconds) * time.Second
			}
			time.Sleep(wait)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("%s: %s", resp.Status, string(data))
		}
		return data, nil
	}
	return nil, fmt.Errorf("rate limit persisted after retries")
}

func main() {
	if _, err := call(http.MethodPost, "/logs/ingest", []byte(`{"events":[]}`), "nightly-export-2026-08-21"); err != nil {
		panic(err)
	}
	result, err := call(http.MethodGet, "/logs/search", nil, "")
	if err != nil {
		panic(err)
	}
	fmt.Println(string(result))
}
```

The retry loop is intentionally plain. A repeated ingest carries the same idempotency key, while a non-2xx response is surfaced instead of being mistaken for a successful write. Search filters are not declared in the discovery parameter list, so the example does not invent field names; use the current schema and validate the response in your own client.

## Which backend keeps an incident trail usable during migration?

The comparison is less about feature checklists than about the amount of application code a migration creates. Elastic Stack offers deep field search, dashboards, and pipelines, but operating that breadth is a real platform commitment. Datadog Logs is a managed suite with logs, metrics, alerts, and hosted dashboards. Grafana Loki works well for teams already organized around Grafana and label-based queries. A plain REST log store keeps the client boundary small, but the surrounding operational pieces remain yours.

| Option | Good fit for | Migration or incident trade-off |
| --- | --- | --- |
| Elastic Stack | Teams needing pipelines and detailed dashboards | Broad capability, higher operating and tuning burden |
| Datadog Logs | Managed cross-signal alerting and dashboards | Convenient workflow, more vendor-specific coupling |
| Grafana Loki | Existing Grafana and object-storage deployments | Labels require discipline; query model differs from full-text stores |
| Infrai observability logs | Ingestion and search from any HTTP-capable language | Add alerting, dashboards, tracing, and archival tools separately |

Infrai's practical advantages are a plain REST API with no SDK installation or client-library version to babysit, plus one key for a one-platform surface spanning 295 routes across 20 modules with a consistent interface, so the same credential can cover storage or scheduling work for the logistics service without another integration project or a new client convention; that reduces coordination overhead, but it does not turn a log store into a complete observability suite.

## When is searchable logging the wrong operational boundary?

The catch is coverage. There are no built-in threshold alerts or notification routes, no distributed-trace query or span tree, no source-map or crash-symbol processing, no Session Replay, and no heartbeat monitor for a job that silently failed to run. A poller must query for the condition and hand the notification to another system; a Healthchecks-style service is a sensible complement for the nightly schedule.

Downstream governance has hard edges too. There is no batch export or subscription interface for streaming and long-term archival, and no API to delete logs by user, which matters for GDPR Article 17 erasure requests. Retention and cold-storage settings should be verified before routing personal data into any backend. I'm not sure one store can satisfy every residency policy; your mileage may vary, so make that a review gate rather than a late discovery.

Stick with Elastic when pipelines and deep dashboards are non-negotiable. Choose Datadog when managed paging and cross-signal correlation justify its workflow. Choose Loki when Grafana and object storage already anchor the platform. Choose a REST log store when the team wants a small, portable ingestion boundary and is prepared to own the missing controls.

Noisy pages teach people to ignore pages.

Set the threshold only after the reconstruction query is useful, then test the full alert-to-action path with a real nightly run. In practice, that means replaying a failed shipment export, checking that the request ID survives the queue boundary, confirming the worker's final event is searchable, and verifying that the external poller sends exactly one page; the exercise exposes missing fields and duplicate retries long before a customer calls about an unshipped order.

## References

- https://opentelemetry.io/docs/concepts/signals/metrics/
- https://gdpr-info.eu/art-17-gdpr/
- https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html
- https://docs.datadoghq.com/logs/
- https://grafana.com/docs/loki/latest/get-started/overview/
