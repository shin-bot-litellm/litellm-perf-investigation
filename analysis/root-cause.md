# Root Cause Analysis

## Architecture Issue

```
┌─────────────────────────────────────────────────────────────┐
│                     STARTUP                                 │
│  Global router_settings → Applied to shared llm_router ✅   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   EVERY REQUEST                             │
│                                                             │
│  1. _get_hierarchical_router_settings()                     │
│     ├─ Check key settings (memory)           ~0ms          │
│     ├─ Query team settings (DB)              ~5-20ms ⚠️    │
│     └─ Query global settings (DB)            ~5-20ms ⚠️    │
│                                                             │
│  2. IF settings found → Create NEW Router                   │
│     └─ Router.__init__                       ~50-500ms 🔥   │
│        └─ PrometheusServicesLogger.__init__                 │
│           └─ is_metric_registered() × 50                    │
│              └─ REGISTRY.collect() [O(n)]    ~10ms each     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Three Compounding Problems

### Problem 1: O(n) Prometheus Scan ✅ FIXED
- `is_metric_registered()` called ~50 times per Router init
- Each call did O(n) iteration over all registered metrics
- With 100+ metrics: ~500ms overhead per request
- **Fix:** PR #20087 - O(1) dict lookup

### Problem 2: Redundant Global Settings 🔄 PR OPEN
- Global `router_settings` applied to shared router at startup
- Same settings fetched again on every request
- Triggers unnecessary per-request Router creation
- **Fix:** PR #20133 - Skip global in hierarchical lookup

### Problem 3: Uncached Team Lookups ⚠️ UNADDRESSED
- Every request with `team_id` queries DB for team settings
- Even empty results aren't cached
- Adds 5-20ms latency per request
- **Proposed:** TTL cache for team router_settings

## Why v1.80.5 Was Fast

The hierarchical router_settings feature was introduced in v1.81.0. In v1.80.5:
- No `_get_hierarchical_router_settings()` call
- No per-request Router creation
- Shared `llm_router` used for all requests

## Metrics

**Before fix (with router_settings in DB):**
- CPU: 800-1000m per pod
- Latency: 30+ seconds
- Timeouts: 60%+

**After removing router_settings:**
- CPU: 100-180m per pod
- Latency: Normal
- Timeouts: Minimal
