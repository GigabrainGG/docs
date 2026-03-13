# Exa-Powered Intelligence Stack

**Date:** 2026-03-12
**Status:** Draft
**Goal:** Replace brittle indexer pipelines with real-time Exa search to fix stale/wrong data that causes bad trading decisions

**Base path:** All file references below are relative to `intelligence-monorepo/agents/brain-core/`

## Problem Statement

Agents make bad trading decisions because their data is stale or wrong. Two confirmed root causes:

1. **Stale macro event data** — The agent told a user "CPI is tomorrow" when CPI had already happened. Root cause: `MacroEventsFetcher` caches events for 1 hour + `MacroPolicy` caches results for 30 min = up to 90 minutes of staleness. A hardcoded fallback always returns "CPI in 3 days" if the DB fetch fails.

2. **Rate limit exhaustion by schedulers** — Token schedulers fire every 5 minutes, generating ~480 CoinGlass API calls per cycle. When combined with autonomous agent cron triggers, this exceeds rate limits. CoinGlass returns empty data silently, and agents trade on incomplete signals. (Deferred to a separate design.)

## Architecture: Current vs Proposed

### Current (Brittle)
```
Playwright scraper → DB table (upcoming_macro_events) → 1-hour in-memory cache
→ 30-min policy result cache → agent gets stale data
```

- Playwright scraper runs on unknown schedule
- DB table freshness depends on external indexer
- Two caching layers compound staleness
- Hardcoded fallback masks failures
- Timezone-naive datetime comparisons produce wrong results

### Proposed (Real-Time)
```
Agent requests macro data → ExaMacroProvider searches Exa API → 15-min Redis cache
→ agent gets fresh data with event results included
```

- Exa search is the primary source of truth
- DB table becomes a warm fallback (not primary)
- Single cache layer with shorter TTL
- Post-event enrichment fetches actual results
- All datetime operations use UTC-aware objects

## Detailed Design

### Component 1: ExaMacroProvider

**New file:** `intelligence/data_services/exa_macro_provider.py`

**Responsibility:** Fetch upcoming and recent macro events via Exa search, parse into structured `CatalystEvent` objects.

**Interface:**
```python
class ExaMacroProvider:
    """Real-time macro event data via Exa search."""

    CACHE_TTL = 900  # 15 minutes

    async def get_upcoming_events(self, days_ahead: int = 14) -> list[dict]:
        """
        Search Exa for upcoming US economic events.
        Returns structured events with: name, date, time_until, expected_impact.
        """

    async def get_event_results(self, event_name: str, lookback_hours: int = 4) -> dict | None:
        """
        Search Exa for actual results of a recently released economic event.
        Returns: actual value, expected value, market reaction summary.
        """
```

**Search queries:**
- Upcoming events: `"US economic calendar {current_week_dates} CPI FOMC NFP PCE GDP jobless claims retail sales"` with `startPublishedDate` set to 7 days ago
- Event results: `"{event_name} {month} {year} results actual vs expected"` with `startPublishedDate` set to lookback window

**Parsing:** Extract event name, date, and metadata from Exa search results. Use LLM-assisted parsing (Claude Haiku 4.5 via existing Anthropic client, ~200ms latency) to convert unstructured search results into structured `CatalystEvent` objects when regex/pattern matching is insufficient. On LLM parsing failure, return raw text summary as a single "upcoming events" catalyst with `expected_impact = "check source"`.

**Caching:** 15-minute Redis cache via the existing `get_api_cache()` infrastructure (consistent with all other data services). Cache key: `exa:macro:{query_hash}`. This ensures cache is shared across workers/processes.

**Exa API access:** Direct `httpx` call to `https://api.exa.ai/search` using `EXA_API_KEY` from environment. This avoids the service proxy overhead and credit deduction (internal infrastructure use, not user-facing). No new SDK dependency needed — raw HTTP like other data services (CoinGlass, DefiLlama).

**Post-event enrichment:** When fetching results for recently-released events, use `asyncio.gather()` to parallelize multiple event lookups (e.g., if CPI and jobless claims released same morning). Max 3 concurrent enrichment calls.

**Export:** Update `intelligence/data_services/__init__.py` to export `ExaMacroProvider` and `get_exa_macro_provider()` singleton, following the existing pattern.

### Component 2: Updated MacroPolicy._get_upcoming_catalysts()

**File:** `intelligence/policies/macro_policy.py`

**Import fix:** Add `UTC` to datetime import: `from datetime import UTC, datetime, timedelta`

**Changes:**

1. **Sole source: ExaMacroProvider** — Remove both LinkUp and DB from the catalyst path entirely.
   ```python
   async def _get_upcoming_catalysts(self) -> list[CatalystEvent]:
       now = datetime.now(UTC)  # Fix: was datetime.now() (timezone-naive)

       # Exa real-time search (sole source — replaces dead DB + LinkUp)
       try:
           provider = get_exa_macro_provider()
           events = await provider.get_upcoming_events(days_ahead=14)
           if events:
               catalyst_events = self._parse_exa_events(events, now)

               # Post-event enrichment (parallel)
               recent_events = [e for e in catalyst_events if -4 <= e.hours_until <= 0]
               if recent_events:
                   results = await asyncio.gather(
                       *[provider.get_event_results(e.event) for e in recent_events],
                       return_exceptions=True
                   )
                   for event, result in zip(recent_events, results):
                       if isinstance(result, dict):
                           event.actual_result = result
               return catalyst_events
       except Exception as e:
           logger.error(f"Exa macro fetch failed: {e}")

       # No fallback. No stale DB. No hardcoded stubs. Just empty list.
       return []
   ```

2. **Remove hardcoded fallback** — Delete the stub `CatalystEvent` list that always returns "CPI Release" with `now + timedelta(days=3)`

3. **Remove LinkUp fallback** — Delete the `get_fed_news_checker().get_upcoming_macro_events()` call from this path. Exa replaces it.

4. **Remove DB fallback** — Delete the `get_macro_events_fetcher().get_upcoming_events()` call from this path. The `upcoming_macro_events` table has been dead for 91 days (last entry: Dec 12, 2025). Not worth keeping as a fallback.

4. **Fix all timezone-naive calls** — Replace every `datetime.now()` with `datetime.now(UTC)` in this file. Known locations:
   - Inside `_get_upcoming_catalysts()` (the `now = datetime.now()` assignment)
   - Inside `analyze()` return statement (`timestamp=datetime.now()`)
   - Any other occurrences found during implementation

### Component 3: Reduced Cache TTLs

| Cache | Before | After | Location |
|-------|--------|-------|----------|
| `@cache_policy_result("macro", ...)` | 1800s (30 min) | 600s (10 min) | `intelligence/policies/macro_policy.py` |
| `MacroEventsFetcher._cache_ttl` | 3600s (1 hour) | N/A (removed from catalyst path) | `intelligence/data_services/macro_events.py` |
| `ExaMacroProvider.CACHE_TTL` | N/A (new) | 900s (15 min) | `intelligence/data_services/exa_macro_provider.py` |

### Component 4: Event Proximity Force-Refresh

When the agent runs and a macro event is within 2 hours (before or after), bypass the policy cache.

The existing `@cache_policy_result` decorator already supports a `use_cache` kwarg (checked at `common/policy_cache.py`). Use this mechanism:

```python
# In the caller that invokes macro analysis (e.g., PolicyRouter.analyze_macro):
async def analyze_macro(self, context):
    # Quick check: do we have a recent event?
    use_cache = True
    cached = self._policy_cache.get("macro")
    if cached and hasattr(cached, 'upcoming_catalysts'):
        if any(abs(e.hours_until) < 2 for e in cached.upcoming_catalysts):
            use_cache = False

    return await self.macro_policy.analyze(context, use_cache=use_cache)
```

This works with the existing decorator — no need for a new `force_fresh` parameter or `_analyze_uncached()` method.

### Component 5: CatalystEvent Enhancement

**File:** `intelligence/indicators/macro/models/MacroPolicyAnalysis.py`

`CatalystEvent` is a Pydantic `BaseModel` (not a dataclass). Add new fields using `Field()`:

```python
class CatalystEvent(BaseModel):
    """Model for upcoming macro catalyst events. All times in UTC."""
    event: str = Field(description="Event name")
    date: datetime | str = Field(description="Event date (UTC)")
    hours_until: float = Field(description="Hours until event (negative = already occurred)")
    days_until: int = Field(description="Days until event")
    time_until_display: str = Field(description="Human-readable time until event")
    expected_impact: str = Field(description="Expected market impact description")
    # New fields:
    actual_result: dict | None = Field(default=None, description="Post-event result: {actual, expected, surprise, market_reaction}")
    data_source: str = Field(default="exa", description="Data source: 'exa' or 'database_fallback'")
    staleness_warning: str | None = Field(default=None, description="Warning if data may be stale")
```

This lets the agent see:
- "CPI released 2 hours ago: 3.2% actual vs 3.1% expected (+0.1% surprise). Markets rallied +0.5%."
- Instead of: "CPI tomorrow" (wrong) or nothing at all

## Files Changed

| File | Change |
|------|--------|
| `intelligence/data_services/exa_macro_provider.py` | **New** — Exa-based macro event fetcher with Redis caching |
| `intelligence/data_services/__init__.py` | Export `ExaMacroProvider` and `get_exa_macro_provider()` |
| `intelligence/policies/macro_policy.py` | Replace `_get_upcoming_catalysts()` with Exa-only, remove DB/LinkUp/hardcoded fallbacks, fix timezone, reduce cache TTL |
| `intelligence/indicators/macro/models/MacroPolicyAnalysis.py` | Add `actual_result`, `data_source`, `staleness_warning` fields to `CatalystEvent` |
| `intelligence/data_services/macro_events.py` | No changes needed (removed from catalyst path, kept for other uses if any) |
| `brain_core/core/config.py` | Add `exa_api_key: str` field (env: `EXA_API_KEY`) |

## Phase 2: News & Alpha via Exa (Future)

Extend the Exa pattern to news and alpha events:

- **Breaking news:** Use Exa to search for token-specific news in the last 24 hours. Supplement (not replace) Turbopuffer historical search.
- **Alpha signals:** Use Exa to detect breaking market-moving events that haven't been indexed yet. Turbopuffer retains its role for structured historical alpha with impact ratings.
- **Hybrid approach:** Exa for "what's happening now" + Turbopuffer for "what happened before and how impactful was it"

## Testing

- **Unit tests:** `ExaMacroProvider` — mock Exa HTTP responses, verify parsing, caching, and error handling
- **Unit tests:** `_get_upcoming_catalysts()` — mock `ExaMacroProvider` and DB fallback, verify cascade logic
- **Integration test:** End-to-end macro analysis with real Exa API (gated behind `@pytest.mark.integration`)
- **Manual verification:** Run agent, confirm it correctly reports CPI status (before/during/after event)

## Success Criteria

1. Agent correctly identifies that CPI has already happened (not "tomorrow") within 15 minutes of the event
2. Agent receives actual event results (X% vs Y% expected) for events that occurred in the last 4 hours
3. No hardcoded fallback data is ever served to agents
4. All datetime comparisons use timezone-aware UTC objects
5. Fallback to DB is clearly marked as stale when used
6. DB data older than 24 hours is rejected (empty list returned instead)

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Exa API down | Return empty catalyst list + log error. Agent proceeds without macro event context (safe — better than fake data) |
| Exa returns irrelevant results | LLM-assisted parsing validates event structure; reject unparseable results; return raw summary as fallback |
| Exa rate limits | 15-min Redis cache prevents excessive calls; shared across workers |
| Parsing errors on event dates | Strict date parsing with fallback to "unknown date" rather than wrong date |
| LLM parsing latency | Claude Haiku 4.5 (~200ms); cached so only on cache miss; total added latency ~1-2s on cold path |
| Cost (Exa) | ~4 calls per 15 min shared across all agents (Redis cache) = negligible (~$0.002/hour) |
| Cost (LLM parsing) | Haiku 4.5 at ~$0.001 per call, only on cache miss = ~$0.004/hour |
| Multiple events same day (N+1) | `asyncio.gather()` parallelizes enrichment, max 3 concurrent |
