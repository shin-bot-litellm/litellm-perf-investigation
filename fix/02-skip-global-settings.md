# Fix #2: Skip Global Router Settings

**PR:** [#20133](https://github.com/BerriAI/litellm/pull/20133)  
**Status:** 🔄 Open  
**Author:** @shin-bot-litellm

## Problem

Global `router_settings` in DB are already applied to shared `llm_router` at startup via `_add_router_settings_from_db_config()`.

But `_get_hierarchical_router_settings()` returns them again, causing:
1. Unnecessary DB query per request
2. Redundant per-request Router creation

## Solution

Remove global settings lookup from hierarchical function:

```diff
  async def _get_hierarchical_router_settings(self, user_api_key_dict, prisma_client):
      # 1. Key-level (keep)
      ...
      
      # 2. Team-level (keep)
      ...
      
-     # 3. Global (remove - already on shared router)
-     db_router_settings = await prisma_client.db.litellm_config.find_first(
-         where={"param_name": "router_settings"}
-     )
-     if db_router_settings:
-         return db_router_settings.param_value
      
      return None
```

## Impact

- Eliminates 1 DB query per request
- Prevents per-request Router creation from global settings
- Users with global `router_settings` won't trigger regression
