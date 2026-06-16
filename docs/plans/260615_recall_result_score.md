---
title: Recall relevance filtering and maintainer feedback
type: investigation
status: closed - raw score exposure rejected upstream
date: 2026-06-15
area: recall API, clients, CLI integrations
---

# Recall relevance filtering and maintainer feedback

## Original problem

`POST /v1/default/banks/{bank_id}/memories/recall` can return memories that are
too weak for downstream prompt injection, but the public `results[]` payload does
not include the internal relevance signal used during retrieval/reranking.

The original plan was to expose a per-result `score` on every recall result so
client integrations could apply a threshold locally, for example dropping memory
results below `0.25`.

## What was implemented on the fork

The fork branch implemented and validated the direct score-exposure approach:

- added a public `score` field to recall results;
- updated generated Python, TypeScript, and Go clients;
- added `recall_score_min` / `recallScoreMin` client-side filtering across the
  Rust CLI and bundled agent integrations;
- defaulted the threshold to `0.25`;
- preserved missing-score results for compatibility with older Hindsight servers;
- preserved an existing saved `recall_score_min` when `hindsight configure`
  updates only API URL/key.

This solved the downstream integration problem, but it exposed an internal ranking
implementation detail as a public API contract.

## Upstream maintainer decision

Upstream rejected the raw score exposure direction in PR feedback:

> please stop working on this - we're not going to expose any score field soon
>
> the score makes only sense if it's relative and exposing one force us to follow
> that algorithm "forever" for backwards compatibility.
>
> client-level threshold are discouraged

Source: https://github.com/vectorize-io/hindsight/pull/2196#issuecomment-4707203632

## Interpretation

The maintainer concern is technically coherent: a raw recall score is not a stable
domain value. It is an output of the current retrieval and reranking pipeline, and
its meaning can shift when Hindsight changes embedding providers, rerankers,
normalization, hybrid search, candidate selection, or score calibration.

If clients receive `score: 0.31`, they will naturally write logic such as
`score >= 0.25`. That turns the numeric value into a long-lived public semantic
promise. Future algorithm improvements could change numeric distributions without
making results worse, but client-side thresholds would still break or silently
change behavior.

The maintainer is therefore drawing a boundary:

- internal scoring can remain an implementation detail;
- public recall quality should be controlled by server-owned semantics;
- clients should not hardcode thresholds against raw ranking math.

## Updated recommendation

Do not pursue public `results[].score` as the upstream path.

The next upstream-friendly design should keep relevance interpretation on the
server side. Reasonable alternatives:

- add a server-owned recall option such as `min_relevance`, `quality`, or
  `strictness`, where Hindsight defines the semantics and maps them to the current
  scoring algorithm internally;
- add coarse labels such as `relevance: "low" | "medium" | "high"` only if
  maintainers are willing to support those labels as stable public semantics;
- improve default server-side recall pruning so weak memories are less likely to
  appear without requiring clients to know about scoring;
- keep raw scores available only in `trace` / debug surfaces with explicit
  non-contract wording.

For downstream integrations, the safest short-term path is to avoid depending on a
public score field until upstream provides a stable server-owned filter.

## Closed PRs

- Fork PR: https://github.com/Korayem/hindsight/pull/1
- Original upstream PR: https://github.com/vectorize-io/hindsight/pull/2196
- Replacement upstream PR: https://github.com/vectorize-io/hindsight/pull/2206

These were closed after the maintainer clarified the API-contract objection.
