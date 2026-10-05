# Recruiting Search Incidents: 6 Source Deduplication Checks Before Retrieval Reranking

In a recruiting candidate search retrieval architecture, deduplicate sources before chunking, retrieve only current canonical records, and rerank afterward. That ordering is the operational recommendation: it prevents repeated profiles from consuming the retrieval window while keeping freshness and provenance available when a recruiter asks why a candidate appeared.

Short answer: for a recruiting marketplace, source deduplication belongs in a durable identity layer between ingestion and retrieval, while chunking, freshness checks, and relevance reranking operate on the canonical candidate record rather than on each imported profile.

I've been woken by alerts that meant nothing and missed the one that mattered. That changes the design question. A polished relevance dashboard isn't evidence that candidate search is healthy; the useful question is, "What page fired when one person occupied four of the ten retrieved positions?" If the answer is "none," the retrieval architecture has an observability gap even when every request returns successfully.

## 1. How should recruiting candidate search retrieval handle source deduplication?

Treat a source record and a candidate as different entities. A source record is an observation from an applicant-tracking feed, a marketplace profile, an import, or another authorized input. A candidate is the canonical search entity to which those observations may resolve. Preserve both, because deleting the source row after a match makes later identity corrections, consent handling, freshness checks, and incident reconstruction much harder.

The identity layer should produce three things: a stable internal candidate ID, the set of source records currently attached to it, and an auditable reason for each link. Exact identifiers that the system is permitted to process can support deterministic links. Weaker signals should produce a reviewable confidence decision, not an irreversible merge disguised as cleanup. I'm not sure any universal threshold is defensible here; the right boundary depends on source quality, jurisdiction, and the cost of joining two different people. A labeled set of known same-person and different-person pairs is what resolves that uncertainty.

This architecture also establishes the unit of retrieval. Embeddings or lexical documents can refer to chunks, but each chunk carries the canonical candidate ID, source provenance, and a freshness marker. Retrieval then collapses by candidate ID before the reranker spends its limited input window. In retrieval-augmented generation, retrieved external memory conditions downstream generation; the same capacity constraint matters here even when the downstream task is a relevance explanation rather than open-ended generation. Duplicate observations are not extra evidence if they merely repeat the same resume sentence.

The catch is that canonicalization is not suitable as a lossy preprocessing shortcut. If legal or operational policy requires each submitted application to remain independently visible, keep those applications as source-level records and deduplicate only the candidate-facing retrieval view. Stick with source-level retrieval when the job is application auditing rather than candidate discovery.

## 2. Check identity before similarity

The first check asks whether deduplication is being used to solve identity or merely visual similarity. Two profiles can share an employer and title without representing the same person; one person can also have a terse marketplace bio and a detailed imported resume that share little text. Vector proximity is useful retrieval evidence, but making it the sole merge key converts a ranking heuristic into an identity verdict.

Use staged decisions. Normalize permitted exact identifiers, block plausible pairs so the system does not compare every record with every other record, score the remaining identity evidence, then send ambiguous pairs to a review or non-merge state. Record the rule version and evidence class with the decision. Long paragraphs matter here because the failure is subtle: once two different candidates are merged, their skills can be interleaved into one retrieval document, the reranker may produce a perfectly plausible score, and an aggregate relevance chart can improve even though the result shown to a recruiter is wrong. A dashboard averages the damage away. A page tied to a sudden rise in contested merges, by contrast, points toward an action.

Don't auto-merge merely because the top score clears a round number.

For a postmortem, ask which invariant failed: one source linked to multiple active canonical candidates, one candidate accumulated mutually exclusive identity evidence, or a merge decision changed without a rule-version change. Those are inspectable conditions. "Search quality declined" is a symptom, not a page.

## 3. Check chunk ownership and freshness

The second check makes every searchable chunk belong to exactly one current canonical candidate and one identifiable source revision. The third rejects stale chunks before reranking. Together they stop an old resume and a new profile from behaving like two votes for the same candidate.

Chunk boundaries should follow information that recruiters search for, such as a role, a project, or a skill-bearing passage, while retaining the candidate ID outside the embedded text. Do not concatenate every source into one giant blob: a small source correction would then invalidate unrelated material, and provenance would be difficult to present. Do not create freestanding chunks without ownership metadata either; after a merge or split, there would be no reliable way to find every affected search document.

A freshness policy needs two clocks. `observed_at` says when the platform received a source revision; `effective_at` says when that revision claims to have become true, when the source supplies such a value. The index should also retain the source revision used to build a chunk. Before reranking, discard or refresh candidates whose indexed revision is no longer current. Your mileage may vary on refresh intervals — a marketplace with recruiter-driven edits has a different tolerance from a nightly authorized import — but the invariant should stay fixed: the serving path must be able to prove which revision produced a result.

Here is a deliberately small Go representation of that boundary. It does not prescribe storage, a vendor, or a similarity model.

```go
package retrieval

import "time"

type Chunk struct {
	ChunkID          string
	CandidateID      string
	SourceID         string
	SourceRevision   string
	Text             string
	ObservedAt       time.Time
	EffectiveAt      time.Time
}

type CandidateState struct {
	CandidateID    string
	CurrentSources map[string]string // source ID to current revision
}

func IsCurrent(chunk Chunk, state CandidateState) bool {
	if chunk.CandidateID != state.CandidateID {
		return false
	}
	revision, ok := state.CurrentSources[chunk.SourceID]
	return ok && revision == chunk.SourceRevision
}
```

It is boring code. Good. At 3am, a direct revision comparison is easier to reason about than a freshness score whose inputs cannot be reconstructed.

## 4. Check collapse order and reranker input

The fourth check verifies that retrieval gathers candidates broadly and then collapses duplicate chunks by canonical candidate ID before relevance reranking. The fifth verifies what evidence survives that collapse. Keep the highest-signal chunks for each candidate, but retain distinct, current evidence when it adds information rather than repeating it. The reranker should receive candidate-level groups with provenance, not a flat list in which repeated sources can manufacture prominence.

Consider a recruiter searching for a Go engineer with marketplace payments experience. Candidate A has the same role copied across an import, a profile, and an application; Candidate B has one current, detailed profile. If the first-stage retriever returns three A chunks and one B chunk, sending that flat sequence downstream lets ingestion frequency compete with relevance. Collapsing first gives the reranker one candidate-level comparison while still allowing A's non-redundant evidence to be represented. No invented benchmark is needed to justify the test: create a fixture where duplicating an unchanged source record cannot improve the candidate's final rank, then make that property part of the release gate.

```go
package retrieval

import "sort"

type Hit struct {
	CandidateID string
	ChunkID     string
	Score       float64
}

func CollapseForReranking(hits []Hit, perCandidate int) []Hit {
	sort.SliceStable(hits, func(i, j int) bool {
		return hits[i].Score > hits[j].Score
	})

	kept := make(map[string]int)
	result := make([]Hit, 0, len(hits))
	for _, hit := range hits {
		if kept[hit.CandidateID] >= perCandidate {
			continue
		}
		result = append(result, hit)
		kept[hit.CandidateID]++
	}
	return result
}
```

This example assumes the caller has already filtered stale chunks and resolved candidate IDs. It also has an explicit limitation: score order alone cannot detect two differently worded chunks that carry the same claim. For richer profiles, add a claim-level or source-revision diversity rule and validate it against labeled queries. Don't quietly pretend a top-`k` cap has understood semantics.

## 5. Check the page, not the dashboard

The sixth check is operational: connect each failure mode to a page with an owner and a runbook. Page on broken invariants or user-visible exhaustion, not on every fluctuation in average duplicate rate. Useful signals include the share of result sets with fewer unique candidates than requested, contested identity links entering the serving index, stale source revisions reaching reranking, and abrupt changes in canonical records per source. Thresholds must come from the marketplace's normal traffic and error budget; inventing universal numbers would create exactly the noisy alerts an incident responder learns to distrust.

Verification should cover offline properties and a guarded production rollout. Build fixtures for exact duplicate imports, weakly matching names, shared employers, a source update, a candidate split, and a candidate merge. Assert that an unchanged duplicate cannot improve rank, stale revisions cannot serve, and a split restores both candidates' independent chunks. Then shadow the new identity map, compare candidate-set changes rather than only score changes, and sample the highest-impact merge decisions with provenance visible.

Rollback deserves design time before deployment. Version the identity mapping and index generation together; keep the previous mapping readable; make derived chunks disposable; and avoid mutating original source observations. If a release links the wrong records, rollback the mapping and rebuild the affected derived documents. Do not attempt to reverse-engineer source history from the search index.

This is also where team cost appears. A sophisticated probabilistic resolver may reduce manual review, but it adds model evaluation, threshold ownership, and on-call diagnosis. A deterministic resolver plus an ambiguous state is slower for some operations but easier to audit. Neither is universally better. Choose the mechanism whose failure can be detected and reversed by the team carrying the pager.

## 6. Run the recovery drill

Before calling the architecture ready, rehearse one merge and one split using production-shaped but non-sensitive fixtures. Confirm that source observations remain unchanged, canonical IDs change through a versioned decision, obsolete chunks leave retrieval, fresh chunks enter it, and reranking sees no duplicate candidate group. Then answer the uncomfortable question: which page fires if only the cleanup step fails?

If nobody can answer without opening a dashboard and guessing, the runbook is incomplete.

The final decision rule is plain: preserve sources, version identity, attach every chunk to a candidate and revision, filter freshness, collapse before reranking, and page on violated invariants. This design will not eliminate ambiguous identity, and it should not. It makes ambiguity visible instead of allowing duplicate ingestion to masquerade as relevance.

## References

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
