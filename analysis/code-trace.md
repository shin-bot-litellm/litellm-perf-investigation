# Code Trace: Per-Request Path

## Overview

```
Request → common_request_processing.py:605
       → _get_hierarchical_router_settings() [2-3 DB queries]
       → route_llm_request.py:198 [new Router if settings found]
       → Router.__init__ → ServiceLogging → PrometheusServicesLogger
       → is_metric_registered() × ~50 [was O(n), now O(1)]
```

## Detailed Trace

### 1. Request Processing Entry
**File:** `litellm/proxy/common_request_processing.py:600-620`

```python
# Apply hierarchical router_settings (Key > Team > Global)
if llm_router is not None and proxy_config is not None:
    from litellm.proxy.proxy_server import prisma_client

    router_settings = await proxy_config._get_hierarchical_router_settings(
        user_api_key_dict=user_api_key_dict,
        prisma_client=prisma_client,
    )

    if router_settings is not None and router_settings:
        model_list = llm_router.get_model_list()
        if model_list is not None:
            user_config = {"model_list": model_list, **router_settings}
            self.data["user_config"] = user_config  # ← triggers per-request Router
```

### 2. Hierarchical Settings Lookup
**File:** `litellm/proxy/proxy_server.py:3382-3479`

```python
async def _get_hierarchical_router_settings(self, user_api_key_dict, prisma_client):
    # 1. Key-level (memory - fast) ✅
    key_router_settings = getattr(user_api_key_dict, "router_settings", None)
    if key_router_settings: return key_router_settings
    
    # 2. Team-level (DB QUERY) ⚠️
    if user_api_key_dict.team_id:
        team_obj = await prisma_client.db.litellm_teamtable.find_unique(
            where={"team_id": user_api_key_dict.team_id}
        )
        if team_obj and team_obj.router_settings:
            return team_obj.router_settings
    
    # 3. Global (DB QUERY) ⚠️
    db_router_settings = await prisma_client.db.litellm_config.find_first(
        where={"param_name": "router_settings"}
    )
    if db_router_settings:
        return db_router_settings.param_value
    
    return None
```

### 3. Per-Request Router Creation
**File:** `litellm/proxy/route_llm_request.py:198-208`

```python
elif "user_config" in data:
    router_config = data.pop("user_config")
    valid_args = litellm.Router.get_valid_args()
    filtered_config = {k: v for k, v in router_config.items() if k in valid_args}
    
    user_router = litellm.Router(**filtered_config)  # NEW ROUTER!
    ret_val = getattr(user_router, f"{route_type}")(**data)
    user_router.discard()
    return ret_val
```

### 4. Router Init Chain
**File:** `litellm/router.py:370-372`

```python
from litellm._service_logger import ServiceLogging
self.service_logger_obj: ServiceLogging = ServiceLogging()
```

**File:** `litellm/_service_logger.py:35-39`

```python
def __init__(self, mock_testing: bool = False) -> None:
    if "prometheus_system" in litellm.service_callback:
        self.prometheusServicesLogger = PrometheusServicesLogger()
```

### 5. Prometheus Init (THE BOTTLENECK - now fixed)
**File:** `litellm/integrations/prometheus_services.py:20-95`

```python
class PrometheusServicesLogger:
    def __init__(self):
        for service in ServiceTypes:  # ~10 services
            # Each service creates 2-3 metrics
            histogram = self.create_histogram(service.value, "latency")
            counter = self.create_counter(service.value, "failed_requests")
            counter = self.create_counter(service.value, "total_requests")
            # Each calls is_metric_registered() → was O(n)!
```

**The Bug (line 107-114, before fix):**
```python
def is_metric_registered(self, metric_name) -> bool:
    for metric in self.REGISTRY.collect():  # O(n) - iterates ALL metrics!
        if metric_name == metric.name:
            return True
    return False
```

**The Fix (PR #20087):**
```python
def is_metric_registered(self, metric_name) -> bool:
    names_to_collectors = getattr(self.REGISTRY, "_names_to_collectors", None)
    if names_to_collectors is not None:
        return metric_name in names_to_collectors  # O(1) dict lookup
    # Fallback for compatibility
    for metric in self.REGISTRY.collect():
        if metric_name == metric.name:
            return True
    return False
```
