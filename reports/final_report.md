# Day 10 Reliability Report

## 1. Architecture summary

`ReliabilityGateway.complete()` ([src/reliability_lab/gateway.py](../src/reliability_lab/gateway.py)) routes every
request through three layers in order: semantic cache, then a per-provider circuit breaker guarding a fallback
chain, then a static degraded response if nothing else works.

```
User Request
    |
    v
[Gateway.complete(prompt)]
    |
    v
[Cache.get(prompt)] ---------------> HIT (score >= threshold, not a false hit)
    |                                     -> return cached text, route="cache_hit:<score>"
    | MISS
    v
[Breaker(primary).call(primary.complete)] --- success --> cache.set(...); route="primary"
    |  raises ProviderError / CircuitOpenError
    v
[Breaker(backup).call(backup.complete)] ----- success --> cache.set(...); route="fallback"
    |  raises ProviderError / CircuitOpenError
    v
[Static fallback] -> "service temporarily degraded", route="static_fallback", error=last_error
```

- **Cache** (`ResponseCache` in-memory, or `SharedRedisCache` over Redis) is checked first so repeated/near-duplicate
  questions never touch a provider at all — zero latency, zero cost, zero risk of tripping a breaker.
- **Circuit breaker** ([circuit_breaker.py](../src/reliability_lab/circuit_breaker.py)) is one instance per provider
  (`self.breakers[provider.name]`), 3-state (`CLOSED → OPEN → HALF_OPEN → CLOSED`), so a failing primary is skipped
  instantly (`CircuitOpenError`) instead of retried until backup takes over.
- **Fallback chain** iterates providers in the configured order; the first successful response wins and is written
  back to cache with `{"provider": <name>}` metadata.
- **Static fallback** guarantees the caller always gets *a* response, with the last real error attached for
  observability, instead of an unhandled exception.

## 2. Configuration

Values taken from [configs/default.yaml](../configs/default.yaml).

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | `record_success()` resets `failure_count` to 0, so this counts *unbroken* failures, not a rolling error rate. At primary's injected `fail_rate=0.25`, three failures in a row without an intervening success has low probability (~1.6%) under normal noise, so the breaker won't trip on background flakiness — but in the `primary_timeout_100` scenario (fail_rate=1.0) it trips after exactly 3 calls, confirmed by `circuit_open_count` jumping from 3–4 in healthy-ish scenarios to 6 in that one (measured via `run_scenario` per-scenario, see §7). |
| reset_timeout_seconds | 2 | Chosen so a full `OPEN → HALF_OPEN → CLOSED` cycle finishes within a single 100-request scenario run (~25–30s wall clock at 180–320ms simulated latency), so the load test can actually *observe* recovery instead of staying OPEN for the whole run. Confirmed: measured `recovery_time_ms` across runs was 2270–2348ms, i.e. one probe round-trip past the 2000ms floor — exactly the expected shape. |
| success_threshold | 1 | A single successful probe fully closes the circuit. This favors fast recovery so oscillation is visible within one test run. Known trade-off: a probe that succeeds by luck immediately re-opens the gate to a still-degraded provider — see §8/§9 for why we'd raise this in production. |
| cache TTL | 300s | Long enough to absorb an entire burst of near-duplicate student questions in one session (a 100-request scenario finishes in under 30s, so 300s covers ~10 such bursts), short enough that a same-day policy edit (e.g., refund deadline correction) doesn't stay served-stale for hours. |
| similarity_threshold | 0.92 | Measured directly with `ResponseCache.similarity()` on the real dataset: `"...refund policy...2024 deadline"` vs `"...refund policy...2026 deadline"` scores **0.970**, and `"tuition fee...2024..."` vs `"...2025..."` scores **0.960** — both *above* 0.92. A threshold alone, even at 0.92, cannot separate these; that's exactly why `_looks_like_false_hit()` exists as a hard guardrail on top of the cosine score (see §5/§8). Meanwhile `"admission FAQ in 5 bullets"` vs `"...in 3 bullets"` scores **0.895** — correctly rejected at 0.92 as a different-shaped request, but would have been a false hit at a looser threshold like 0.85. 0.92 was picked as the highest cutoff that still treats `"Explain circuit breaker states..."` vs `"...retry and circuit breaker patterns"` (0.400) as a clear miss. |
| load_test requests | 100 | Large enough for a meaningful P95/P99 (300 samples across 3 scenarios) and enough volume to let the breaker cycle open→closed more than once per scenario; small enough that the full 3-scenario chaos run finishes in well under a minute, keeping the edit-run-observe loop fast while tuning the above values. |

## 3. SLO definitions

Actual values from the baseline run in [reports/metrics.json](metrics.json) (cache enabled, memory backend, seed-free
random run — see §4 for the full dump).

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 97.33% | **No** |
| Latency P95 | < 2500 ms | 314.44 ms | Yes |
| Fallback success rate | >= 95% | 90.0% | **No** |
| Cache hit rate | >= 10% | 56.33% | Yes |
| Recovery time | < 5000 ms | 2308.27 ms | Yes |

Both misses trace to the same root cause: `backup` also has a non-zero `fail_rate` (0.05) and its own breaker
(`failure_threshold=3`). In the `primary_timeout_100` scenario, primary is *always* down, so every one of its 100
requests depends on backup alone; a 5%-per-call failure rate has a non-trivial chance of landing 3 in a row over 100
trials, which briefly opens the backup breaker too and forces a `static_fallback`. That single-provider dependency —
not a bug in the breaker logic — is what pulls availability and fallback-success below target. See §8.

## 4. Metrics

Pasted directly from `reports/metrics.json` (baseline: `configs/default.yaml`, cache enabled, memory backend, 300
requests across 3 scenarios).

| Metric | Value |
|---|---:|
| availability | 0.9733 |
| error_rate | 0.0267 |
| latency_p50_ms | 268.99 |
| latency_p95_ms | 314.44 |
| latency_p99_ms | 319.33 |
| fallback_success_rate | 0.9 |
| cache_hit_rate | 0.5633 |
| estimated_cost_saved | 0.169 |
| circuit_open_count | 9 |
| recovery_time_ms | 2308.27 |

## 5. Cache comparison

Two full 300-request runs against the same scenarios/providers, only `cache.enabled` differs
(`configs/no_cache.yaml` vs `configs/default.yaml`).

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | 274.44 | 268.99 | -5.45 ms (-2.0%) |
| latency_p95_ms | 316.21 | 314.44 | -1.77 ms (-0.6%) |
| estimated_cost | 0.123624 | 0.056594 | **-0.067030 (-54.2%)** |
| cache_hit_rate | 0.0 | 0.5633 | +0.5633 |

The latency percentiles barely move even though 56% of requests are cache hits — that's because
`run_scenario()` only appends to `latencies_ms` when `result.latency_ms > 0` (cache hits report `latency_ms=0` by
design), so P50/P95/P99 here measure *provider-call* latency specifically, not blended end-to-end latency. The real
win shows up in **cost** (54% lower) and in the knock-on reliability effect: `circuit_open_count` was **20** without
cache vs **9** with it in this pair of runs — the cache absorbs a large share of traffic before it ever reaches the
flaky primary, so the breaker sees fewer real calls and trips less often.

## 6. Redis shared cache

- **Why in-memory cache is insufficient for multi-instance deployments:** `ResponseCache._entries` is a plain Python
  list living in one process's memory. If the gateway runs as 3 pods behind a load balancer, each pod builds its own
  `ResponseCache` via `build_gateway()` — pod A caching an answer does nothing for pods B and C, so the *effective*
  hit rate across the fleet drops roughly 3x for no reason, and identical questions keep re-hitting paid providers.
- **How `SharedRedisCache` solves this:** it stores each entry as a Redis hash (`{"query", "response"}`) under
  `rl:cache:<md5(query)>` with `EXPIRE` for TTL, so every pod reads/writes the same keyspace — a cache write from any
  instance is immediately visible to all others.

### Evidence of shared state

Two independent `SharedRedisCache` Python objects (simulating two gateway pods), same Redis, same prefix:

```
instance_a.set(...) called on process/object A
instance_b.get(...) (different object, same Redis) -> cached='Tuition for 2025 is 120,000,000 VND.', score=1.0
```

`instance_b` never called `.set()` — it only saw the entry because both instances point at the same Redis backend.
This mirrors `tests/test_redis_cache.py::test_shared_state_across_instances`, which passes (6/6 Redis tests green
with `docker compose up -d` running).

### Redis CLI output

```bash
$ docker exec k3-day25-track3-reliability-agent-redis-1 redis-cli KEYS "rl:cache:*"
rl:cache:9e413fd814eb
rl:cache:d354658dc020
rl:cache:fff10da1c72c
rl:cache:da61fb49b4f6
rl:cache:3dab98c0e49e
rl:cache:4fc3c69b9376
rl:cache:3936614ac4c2
rl:cache:dacb2b833659
rl:cache:98332d0d1c9c
rl:cache:844ef0143a5c
rl:cache:734852f3cf4a
rl:cache:8baa2cfa11fa
rl:cache:0bc3b1acf73d
rl:cache:095946136fea
(14 keys, populated by one chaos run against configs/redis.yaml)

$ docker exec ... redis-cli HGETALL rl:cache:8baa2cfa11fa
response "[backup] reliable answer for: Explain how half-open state prevents retry storms."
query    "Explain how half-open state prevents retry storms."
```

### In-memory vs Redis latency comparison (optional)

Same 300-request/3-scenario run, only the cache backend differs (`configs/default.yaml` vs `configs/redis.yaml`).

| Metric | In-memory cache | Redis cache | Notes |
|---|---:|---:|---|
| latency_p50_ms | 268.99 | 285.32 | ~16ms (6%) higher — `SCAN`-based similarity lookup over the network vs an in-process list |
| latency_p95_ms | 314.44 | 318.30 | ~4ms higher, within noise given 180–320ms simulated provider latency dominates the tail |

## 7. Chaos scenarios

Per-scenario numbers below come from calling `run_scenario()` directly for each scenario in isolation (not the
combined `metrics.json`), so they aren't diluted by the other two scenarios.

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | `circuit_open_count=6`, 97% availability, 93% fallback-success; 3/100 requests hit `static_fallback` when backup itself had a bad streak | **Pass** — matches expected shape, static fallback rate small and explained (§3) |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | `circuit_open_count=3`, `recovery_time_ms≈2329` (one full open→half-open→closed cycle observed), 98% availability | **Pass** |
| all_healthy | All requests via primary, no circuit opens | `circuit_open_count=4`, 34/100 responses served via fallback | **Fail** — see note below |
| primary_flaky_50 vs primary_timeout_100 (cache pressure) | Cache absorbs enough traffic to reduce real breaker trips | Combined-run `circuit_open_count` dropped from 20 (no cache) to 9 (with cache) across the same 3 scenarios | **Pass** (custom scenario, see §5) |

**Root cause for `all_healthy` failing its own name:** `provider_overrides: {}` means *no* override is applied —
the scenario silently falls back to the base config's `primary.fail_rate: 0.25`, `backup.fail_rate: 0.05`. It never
actually sets fail rates to 0, so "all healthy" is really "default noise levels" and legitimately trips the breaker
a few times. This is a starter-config naming bug, not a circuit-breaker bug — captured as a next step in §9.

## 8. Failure analysis

**Weakness: `SharedRedisCache` has no graceful-degradation path — a Redis outage takes down the entire gateway, not
just the cache.**

Looking at [cache.py](../src/reliability_lab/cache.py), `ping()` is the *only* method that catches Redis exceptions.
`get()` and `set()` call `self._redis.hget/hset/expire/scan_iter` directly with no `try/except`. In
`ReliabilityGateway.complete()`, the only exceptions caught are `ProviderError` and `CircuitOpenError` — a
`redis.exceptions.ConnectionError` raised from `self.cache.get(prompt)` propagates straight out of `complete()`
uncaught. So if Redis goes down while `cache.backend: redis`, every single request fails with an unhandled exception
— even if both providers are perfectly healthy. The circuit breaker, which is supposed to be the reliability layer
here, never even gets a chance to run, because the crash happens one line earlier, at the cache check.

**Proposed fix:** treat cache lookups as *best-effort*, the same way `ping()` already does:
1. In `SharedRedisCache.get()`/`set()`, catch `redis.exceptions.RedisError` and degrade to a cache-miss (`return
   None, 0.0`) / no-op write, logging a warning — mirroring the existing `ping()` pattern instead of adding new
   exception-handling philosophy.
2. Optionally wrap the Redis-backed cache with a small local `ResponseCache` as an L1 fallback so a Redis outage
   degrades to "slightly less shared caching" instead of "zero caching," without needing changes in `gateway.py` at
   all (the interface — `get`/`set` returning `tuple[str | None, float]` — already matches).

This keeps the fix scoped to the cache layer, consistent with the "cache is optional, providers are required"
design already implied by `cache: ResponseCache | SharedRedisCache | None = None` in `ReliabilityGateway.__init__`.

## 9. Next steps

1. **Redis graceful degradation** (from §8): catch `RedisError` in `SharedRedisCache.get()`/`set()` and fall back to
   a cache-miss instead of raising, so an optional layer can never crash the required path.
2. **Fix the `all_healthy` scenario** in `configs/default.yaml` to actually zero out fail rates
   (`provider_overrides: {primary: 0.0, backup: 0.0}`) instead of relying on `{}`, so it's a real healthy baseline
   instead of "default noise" that still trips the breaker 4 times (§7).
3. **Replace the trivial pass/fail criterion** in `run_simulation()` (currently `successful_requests > 0`, flagged
   by its own `# TODO(student): Define pass/fail criteria per scenario` comment) with real per-scenario thresholds —
   e.g. `primary_timeout_100` should require `fallback_success_rate > 0.9`, not just "at least one success out of
   100 requests."
