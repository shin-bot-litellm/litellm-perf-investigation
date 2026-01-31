# LiteLLM Performance Regression Investigation

**Issue:** [BerriAI/litellm#19921](https://github.com/BerriAI/litellm/issues/19921)  
**Versions Affected:** v1.81.0, v1.81.3, v1.80.13+  
**Last Working:** v1.80.5

## Summary

Performance regression caused by new hierarchical `router_settings` feature introduced in commit `51759424a6`:

1. **2-3 DB queries per request** to check for router settings (team lookup + global settings)
2. **No caching** for router_settings queries (unlike other auth-related lookups)
3. **New Router instance per request** when any settings found in DB
4. **O(n) Prometheus scan** on Router init (fixed by PR #20087)

## Key Finding: Empty Config Still Affected

Users with **empty** `LiteLLM_Config` and `LiteLLM_TeamTable` tables still experience slowdown because:
- DB queries are made on **every request** regardless of table contents
- Router settings queries are **not cached**
- Each query adds 1-5ms network latency

See `evidence/empty-config-case.md` for detailed analysis.

## Status

| Issue | Status | Fix |
|-------|--------|-----|
| O(n) Prometheus scan | ✅ Fixed | [PR #20087](https://github.com/BerriAI/litellm/pull/20087) |
| Redundant global settings | 🔄 Open | [PR #20133](https://github.com/BerriAI/litellm/pull/20133) |
| Uncached router_settings queries | ⚠️ Proposed | See `fix/04-cache-router-settings.md` |
| Empty table overhead | ⚠️ Proposed | See `evidence/empty-config-case.md` |

## Root Cause

**Commit:** `51759424a6` ("Key and Team Routing Setting")  
**Introduced In:** v1.80.13.rc.1  
**Function:** `_get_hierarchical_router_settings` in `proxy_server.py`

The function queries the database on every request to check:
1. Key-level router_settings (from already-loaded object - no DB hit)
2. Team-level router_settings (`litellm_teamtable.find_unique`) - **DB query**
3. Global router_settings (`litellm_config.find_first`) - **DB query**

Unlike other auth lookups that use `DualCache`, these queries hit the database directly.

## Contents

- `evidence/` - User reports, symptoms, and empty-config analysis
- `analysis/` - Code traces and root cause
- `fix/` - Proposed solutions

## Quick Links

- [Empty Config Case Analysis](evidence/empty-config-case.md)
- [Original Issue #19921](https://github.com/BerriAI/litellm/issues/19921)
- [PR #20087 - Prometheus Fix](https://github.com/BerriAI/litellm/pull/20087)
- [PR #20133 - Global Settings Fix](https://github.com/BerriAI/litellm/pull/20133)
