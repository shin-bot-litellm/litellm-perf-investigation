# LiteLLM Performance Regression Investigation

**Issue:** [BerriAI/litellm#19921](https://github.com/BerriAI/litellm/issues/19921)  
**Versions Affected:** v1.81.0, v1.81.3  
**Last Working:** v1.80.5

## Summary

Performance regression caused by new hierarchical `router_settings` feature:
1. **2-3 DB queries per request** to check for router settings
2. **New Router instance per request** when any settings found in DB
3. **O(n) Prometheus scan** on Router init (fixed by PR #20087)

## Status

| Issue | Status | Fix |
|-------|--------|-----|
| O(n) Prometheus scan | ✅ Fixed | [PR #20087](https://github.com/BerriAI/litellm/pull/20087) |
| Redundant global settings | 🔄 Open | [PR #20133](https://github.com/BerriAI/litellm/pull/20133) |
| Uncached team DB queries | ⚠️ Proposed | See `fix/03-cache-team-settings.md` |

## Contents

- `evidence/` - User reports and symptoms
- `analysis/` - Code traces and root cause
- `fix/` - Proposed solutions
