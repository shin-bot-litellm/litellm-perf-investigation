# Fix #4: Optional Feature Flag

**Status:** 💡 Suggestion  
**PR:** Not yet created

## Problem

Most users don't use per-key/per-team router_settings. They pay the cost of DB lookups for a feature they don't use.

## Proposed Solution

Add config flag to disable hierarchical router_settings entirely:

```yaml
# config.yaml
general_settings:
  enable_hierarchical_router_settings: false  # Default: true for backward compat
```

```python
# common_request_processing.py
if llm_router is not None and proxy_config is not None:
    # Check feature flag
    if general_settings.get("enable_hierarchical_router_settings", True):
        router_settings = await proxy_config._get_hierarchical_router_settings(...)
        if router_settings:
            user_config = {"model_list": model_list, **router_settings}
            self.data["user_config"] = user_config
```

## Impact

- Zero overhead for users who don't need the feature
- Opt-in for users who need per-key/per-team settings
- Clear performance vs flexibility tradeoff
