# Issue Breakdown

From @AlexsanderHamir's triage:

| # | Cause | Reporter | Status |
|---|-------|----------|--------|
| 1 | `router_settings` in DB → per-request Router init + Prometheus scan | runixer | ✅ Fixed (PR #20087) |
| 2 | Slowness with **empty** router_settings | superpoussin22 | ⚠️ Unresolved |
| 3 | `/v1/embeddings` 100% CPU | elluvium | ⚠️ Unresolved |
| 4 | Management API / login timeouts | phanisarman | Likely overlaps #1/#2 |

## Issue #2 Analysis

superpoussin22 reports slowness even with empty router_settings tables.

**Hypothesis:** The DB queries themselves add latency even when returning empty results:
- Team lookup: `find_unique()` on every request with team_id
- Global lookup: `find_first()` on every request

Both queries run on EVERY request regardless of whether settings exist.

## Issue #3 Analysis

elluvium reports embeddings-specific CPU issues.

**Hypothesis:** Embeddings typically have higher request volume (batch processing), amplifying the per-request overhead. May not be a separate issue - just more visible due to load pattern.
