# Fix #1: Prometheus O(1) Lookup

**PR:** [#20087](https://github.com/BerriAI/litellm/pull/20087)  
**Status:** ✅ Merged  
**Author:** @AlexsanderHamir

## Problem

`is_metric_registered()` used `REGISTRY.collect()` which iterates all metrics (O(n)).
Called ~50 times per Router init, creating O(n×50) overhead per request.

## Solution

Use `REGISTRY._names_to_collectors` dict for O(1) lookup:

```python
def is_metric_registered(self, metric_name) -> bool:
    # O(1) path
    names_to_collectors = getattr(self.REGISTRY, "_names_to_collectors", None)
    if names_to_collectors is not None:
        return metric_name in names_to_collectors
    
    # Fallback for registries without _names_to_collectors
    for metric in self.REGISTRY.collect():
        if metric_name == metric.name:
            return True
    return False
```

## Impact

- Router init time: ~500ms → ~5ms
- CPU under load: Significant reduction
- Per-request overhead: Eliminated for Prometheus path
