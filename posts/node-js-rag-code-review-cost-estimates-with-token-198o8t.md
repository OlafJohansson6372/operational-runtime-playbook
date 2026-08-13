# Node.js RAG Code Review: Cost Estimates with Token Counts, Embeddings, and Semantic Search

Short answer: for an ask-your-docs code-review service, estimate document indexing and review-time generation separately, then choose the smallest retrieved context that still produces valid structured findings. Batch embeddings can make a backfill easier to budget, but output validation is the gate that decides whether the design is useful.

The experiment is easy to describe. Feed a retrieval system an e-commerce repository, ask it to review a change, and require findings with a file, line range, severity, and explanation. A plain “retrieve some chunks and ask for a review” prompt may sound fine while returning prose that cannot be imported into a pull-request workflow. The cost estimate is only meaningful after that contract is tested.

The contract comes first.

## What does a useful estimate include?

Treat indexing as a corpus workload and review generation as a query workload. For indexing, count the cleaned source text, chunk overlap, metadata that is actually embedded, and the number of documents changed by the backfill. For a review, count the query representation, retrieved chunks, instructions, structured-output overhead, and generated findings. File megabytes are not token counts.

This distinction matters because the two paths repeat differently. A repository re-index may happen after a release or parser change; review-time prompts recur for every pull request. A large embedding batch can dominate one maintenance window, while four extra retrieved files can dominate the recurring review budget. Keep both numbers visible.

Here is a deliberately small ledger. The rates are inputs, not a price claim, and the sample values are test fixtures rather than a benchmark. I would fill the rates from the selected service's current pricing page only after the evaluation set is fixed.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class ReviewWorkload:
    changed_documents: int
    index_tokens: int
    reviews: int
    prompt_tokens_per_review: int
    output_tokens_per_review: int


def estimate_usd(
    workload: ReviewWorkload,
    embedding_usd_per_million: float,
    input_usd_per_million: float,
    output_usd_per_million: float,
) -> dict[str, float]:
    million = 1_000_000
    index_cost = (
        workload.index_tokens * embedding_usd_per_million / million
    )
    review_cost = workload.reviews * (
        workload.prompt_tokens_per_review * input_usd_per_million
        + workload.output_tokens_per_review * output_usd_per_million
    ) / million
    return {
        "index_cost_usd": round(index_cost, 6),
        "review_cost_usd": round(review_cost, 6),
        "total_cost_usd": round(index_cost + review_cost, 6),
    }


trial = ReviewWorkload(
    changed_documents=240,
    index_tokens=480_000,
    reviews=2_000,
    prompt_tokens_per_review=1_100,
    output_tokens_per_review=220,
)
print(estimate_usd(trial, 0.0, 0.0, 0.0))
```

The `changed_documents` field is not used in the arithmetic. That is intentional: it remains an audit dimension so an estimate can say which corpus snapshot it represents. A ledger that reports 480,000 tokens without the files, parser version, and chunking rule behind it is hard to reproduce.

I also keep a separate line for failed validation and retries. A response that is cheap per successful call can become expensive if the application silently asks the model to repair malformed JSON. Your mileage may vary by tokenizer and traffic shape, so I’m not sure a monthly estimate is credible until it includes the observed distribution of chunk sizes and review lengths.

## How should a Node.js RAG budget count tokens before semantic search?

The retrieval pipeline should have an explicit accounting boundary: parse the changed files, remove irrelevant generated output, split by code structure where possible, embed the chunks, and record the chunk IDs. At question time, embed the review request and changed-file summary, search the vector index, and assemble a bounded prompt from the selected chunks. Each transition gets a trace field.

For an online code review, I would record at least:

- repository and commit identifiers;
- changed file paths and chunk IDs;
- index tokens and query tokens;
- retrieved and finally included chunks;
- input and output tokens for the reviewer;
- validation result, retry count, and time to result.

That ledger exposes the failure mode that averages hide. Suppose a checkout change retrieves the entire payment module because a shared helper has similar names. The answer may be plausible, but the prompt is larger and the evidence is less focused. Lowering `top_k` is not automatically the fix; a retrieval test must show that the relevant chunk still appears and that the structured finding remains correct.

Here is the concrete trap I would put in the eval harness. A changed tax helper imports a currency formatter, and the vector search returns a nearby checkout serializer with the same word, “amount.” The reviewer then emits a valid object pointing at the serializer, so a schema check passes and the token ledger looks healthy. A human comparison against the diff fails it. To catch that case, the test record needs the expected file and line range, the relevant chunk rank, the final context, and the model output before any repair step. I would run the same change with `top_k` values of 2, 4, and 8, then compare recall, valid-object rate, prompt tokens, and false-positive rate. The winning configuration is not the one with the smallest prompt; it is the smallest prompt that preserves the correct evidence and does not turn a harmless refactor into a blocking comment. That result is a decision artifact, not a universal benchmark.

The practical cost lever is usually context selection, not deleting fields from the finding. A finding without a stable location is cheaper to generate but unusable in a pull-request comment. Measure the smallest context that preserves the required fields, severity classification, and evidence quote.

## Can structured findings make the cheaper path the wrong choice?

Yes. A code-review result is an interface, not a paragraph. Define the contract before comparing token totals. The example below is a generic validator for a JSON-like Python value; it does not depend on a particular model SDK.

```python
from typing import Any


REQUIRED_FIELDS = {
    "file": str,
    "start_line": int,
    "end_line": int,
    "severity": str,
    "message": str,
    "evidence": str,
}
ALLOWED_SEVERITIES = {"low", "medium", "high", "critical"}


def validate_findings(value: Any) -> list[str]:
    errors: list[str] = []
    if not isinstance(value, dict) or not isinstance(value.get("findings"), list):
        return ["response must contain a findings list"]

    for index, finding in enumerate(value["findings"]):
        if not isinstance(finding, dict):
            errors.append(f"findings[{index}] must be an object")
            continue
        for field, expected_type in REQUIRED_FIELDS.items():
            if not isinstance(finding.get(field), expected_type):
                errors.append(f"findings[{index}].{field} has the wrong type")
        if finding.get("severity") not in ALLOWED_SEVERITIES:
            errors.append(f"findings[{index}].severity is not allowed")
        if (
            isinstance(finding.get("start_line"), int)
            and isinstance(finding.get("end_line"), int)
            and finding["start_line"] > finding["end_line"]
        ):
            errors.append(f"findings[{index}] has a reversed line range")
    return errors


sample = {
    "findings": [
        {
            "file": "checkout/tax.py",
            "start_line": 42,
            "end_line": 45,
            "severity": "high",
            "message": "The discount is applied before the tax jurisdiction check.",
            "evidence": "tax_total = subtotal - discount",
        }
    ]
}
print(validate_findings(sample))
```

The validator catches shape errors, not truth. A finding can have valid types and still be hallucinated, point at the wrong line, or miss a regression. That is why my eval set would contain labeled changes: missing authorization on an order endpoint, a currency conversion mistake, and a harmless refactor that should produce no finding. The last case matters. A system that reports something on every diff can look thorough while damaging reviewer trust.

I include a synthetic `422` validation case in the harness, too. It checks that malformed output is rejected and counted as a failure rather than silently repaired into a confident comment. I've found that this boundary makes cost discussions clearer: retries are an observable consequence of contract quality, not an invisible implementation detail.

## Where do batching, latency, and privacy change the decision?

Batch embedding belongs on the ingestion side when a repository snapshot is large and freshness can wait. It can reduce coordination overhead and make a backfill easier to monitor, but it does not make the interactive review path faster. A new pull request still needs a query embedding, retrieval, generation, and validation. Measure time-to-searchable for changed files separately from time-to-review-result.

For a small repository with frequent edits, a simple queue and direct indexing calls may be easier to reason about than a batch workflow. For a large monorepo, batching can be a better operational fit, provided the ledger records partial completion and the exact snapshot associated with each vector. The catch is that a cheap batch does not justify stale code context.

Privacy is part of the architecture. Repository chunks can contain customer identifiers, order details, or payment-adjacent logic. Minimize what enters logs, separate content from identifiers, define retention, and make access decisions explicit. If the repository includes regulated health information, the applicable security and privacy obligations need a separate review; a retrieval design cannot treat an embedding as automatically anonymous. The HIPAA rules are a useful reference point for that kind of boundary, even though an ordinary code-review corpus may not fall under them.

## What should pass before this reaches production?

I would require four gates: retrieval recall on labeled changes, structured-output validity, grounded finding accuracy, and a cost/latency report tied to a corpus snapshot. Add a no-finding set and malformed-output cases. Track p50 and tail latency, but keep the decision rule boring: use the least expensive configuration that clears the quality gates.

The design is not suitable when the team needs a provider-specific control that the generic runtime cannot expose, or when policy forbids sending source code to an external service. Stick with a self-hosted embedding and generation path when those constraints decide the architecture. Use a managed path when operational simplicity matters more, but retain the same token ledger and contract tests; moving infrastructure does not remove the evaluation problem.

Three words: validate the shape.

Then measure the truth.

## References

- [Function calling guide](https://platform.openai.com/docs/guides/function-calling)
- [45 CFR Part 164, HIPAA Security and Privacy Rules](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)
