# User Reports from Issue #19921

## Impact Summary

| User | CPU Impact | Latency Impact | Endpoint |
|------|-----------|----------------|----------|
| phanisarman | No spike | 300ms → 30s+, 60% timeouts | UI + LLM |
| mishaja12 | Significant spike | - | General |
| elluvium | 100% CPU | - | /v1/embeddings |
| superpoussin22 | High | LLM latencies | General |
| runixer | 800-1000m → 100-180m | - | General |

## Key Quotes

### phanisarman (original reporter)
> "50 Internal users with 200 models connected to postgres, 10 RPS. callbacks to langfuse OTEL. shuffle routing."
> 
> "The entire UI experience was slow... user/list API typically takes around 300 ms in 1.80.5, but it was much slower in 1.81.x. Some management APIs were taking up to 30 seconds"
>
> "Actually all API calls are slow, after a few requests, >60% of API calls are timing out, some LLM calls taking 200s"

### elluvium
> "This is definitely not just a UI issue. We have observed significant performance degradation when using the /v1/embeddings endpoint to index data sources. This process loads the CPU to 100%, which throttles other requests"

### runixer (root cause discovery)
> "py-spy profile showed ~25% of CPU time in `route_request → Router.__init__ → prometheus` path"
>
> "After removing the stale router_settings and restarting pods, CPU usage dropped significantly (from 800-1000m to ~100-180m per pod under load)"

### superpoussin22
> "On prod everything is empty on my side and I can see the issue"
> 
> "STORE_MODEL_IN_DB=True is set on my side and 90% of the models are in config files"

## Workaround Confirmed by runixer

```sql
-- Check for global router_settings
SELECT param_name, param_value FROM "LiteLLM_Config" WHERE param_name = 'router_settings';

-- Check for team router_settings
SELECT team_id, router_settings FROM "LiteLLM_TeamTable"
WHERE router_settings IS NOT NULL AND router_settings != '{}';

-- Remove if not needed
DELETE FROM "LiteLLM_Config" WHERE param_name = 'router_settings';
```
