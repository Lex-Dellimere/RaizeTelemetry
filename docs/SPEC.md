# telemetry (`tm_`): API spec v0.2 (approved by raize_web 2026-11-05)

**Release plan:** v0.1 ships events, handlers, spans and the clock (§1–§4), which is all raize INSTR needs. Metrics (§5) follow in v0.2.

A generic, dependency-free Mettle package for named events, handlers, spans and metrics, in the style of Elixir `:telemetry` and `telemetry_metrics`. The mbuild package name is `telemetry`, so imports look like `import "telemetry/telemetry"`. The public prefix is `tm_` / `Tm` / `TM_`, and internals use `tm__`. The package uses Mettle stdlib only: `std/thread` mutex. It has no exporters.

Sources: raize_web reports/030, `.apex/DECISIONS.md` D8, and raize_web review 2026-11-05.

## 0. Constraints this design works around

| Constraint | Source | Consequence here |
|---|---|---|
| No host atomics | Mettle reports/008 | Vendor `@naked` x86-64 asm atomics (same technique as `alph/src/sys/atomic.mettle`) in `src/sys/atomic.mettle`. Swap to built-ins when 008 lands. |
| No `std/time` | roadmap item 3 / Mettle 018 | Raw `clock_gettime` extern in `src/sys/clock.mettle`. Swap when std/time lands. |
| Indirect-call ICE `call_indirect_unsupported` | raize reports/016 | Handlers get exactly two args, `(TmEvent*, rawptr)`. Fn pointers are stored typed, never cast through rawptr. Values are a plain struct, not a tagged enum passed by value. |
| Fragile generics | raize reports/014, 015 | No maps or generics. KV arrays are pointer + count. |

**v0.1 platform:** Linux x86-64 only, because of the asm and the clock.

## 1. Values

```
const TM_INT = 0; TM_F64 = 1; TM_BOOL = 2; TM_STR = 3;

struct TmValue { kind: int32, i: int64, f: float64, s: string }  // s is a view; i also holds bool (0/1)
struct TmKV    { key: string, val: TmValue }

fn tm_int(key: string, v: int64) -> TmKV
fn tm_f64(key: string, v: float64) -> TmKV
fn tm_bool(key: string, v: bool) -> TmKV
fn tm_str(key: string, v: string) -> TmKV
fn tm_kv_find(kvs: TmKV*, n: int32, key: string) -> TmKV*   // null if absent; linear scan
```

**Ownership:** the caller owns `meas` and `meta` arrays, usually on its stack. They are borrowed for the duration of the `tm_execute` call only. A handler that keeps any key, value or string must copy it.

## 2. Events and handlers

```
// Event ids are plain int32: an index into the live table; < 0 = invalid or an error code.

struct TmEvent {
  id: int32, name: string,               // name points into the table; valid until tm_shutdown
  meas: TmKV*, n_meas: int32,
  meta: TmKV*, n_meta: int32,
}

type TmHandlerFn = fn(ev: TmEvent*, cfg: rawptr) -> void;

fn tm_init() -> int32            // main thread, before any other tm_ call and before starting threads; idempotent (later calls return TM_OK)
fn tm_shutdown()                 // after all threads stop; frees tables, retired tables, names

fn tm_event(name: string) -> int32
fn tm_attach(handler_id: string, prefix: string, f: TmHandlerFn, cfg: rawptr) -> int32
fn tm_attach_many(handler_id: string, prefixes: string*, n: int32, f: TmHandlerFn, cfg: rawptr) -> int32
fn tm_detach(handler_id: string) -> int32
fn tm_execute(id: int32, meas: TmKV*, n_meas: int32, meta: TmKV*, n_meta: int32)
fn tm_has_handlers(id: int32) -> bool   // one acquire load; use it to skip building KVs on hot paths
```

**Return codes:** `TM_OK = 0`, `TM_ERR_EXISTS = -1` (handler_id already attached), `TM_ERR_NOT_FOUND = -2`, `TM_ERR_UNINIT = -3`, `TM_ERR_FULL = -4`, `TM_ERR_ARG = -5` (n <= 0 or a null fn).

### Semantics
- **Event names** are dotted strings, like `raize.request.stop`. `tm_event` is idempotent: the same name returns the same id. Register events at startup and keep the ids in globals. Names are copied.
- **Prefix match is by whole segment.** Prefix `raize.request` matches `raize.request` and `raize.request.stop`. It does not match `raize.requests`. The empty prefix `""` matches every event.
- **Matching happens when the table is built, not when an event fires.** Every `tm_event`, `tm_attach`, `tm_attach_many` and `tm_detach` rebuilds the event → handlers lists from the full attachment list. An event registered after an attach is therefore matched against the existing prefixes.
- **Handler order** for an event is attach order. A handler attached under several matching prefixes runs once per event.
- `tm_execute` runs the handlers synchronously on the caller's thread. An id that is invalid or has no handlers returns immediately.
- **`cfg` must stay valid until `tm_shutdown`.** After `tm_detach` returns, an execute on another thread may still call the handler through the old table. Detaching is therefore not a point where `cfg` can be freed. The same holds for metrics.
- **Handlers must be thread-safe** and must not fail. Mettle has no catch, so there is no auto-detach like `:telemetry`'s. A handler may call `tm_execute`, `tm_attach` and `tm_detach`. Changes take effect on the next execute, because the running execute holds the old table.

### Concurrency: copy-on-write table
- `TmTable` is immutable once published. It holds the event names, a per-event handler array (fn and cfg, typed), and the attachment list.
- **Writers** (`tm_event`, `tm_attach*`, `tm_detach`) take one mutex (`std/thread` `mutex_lock_infinite`), build a new table, publish it with `tm__atomic_store_sc(&g_table, new)`, and push the old table onto a retired list. Retired tables are freed only at `tm_shutdown`, because writes are rare and mostly at startup. Bounded leak is acceptable.
- **`tm_execute`** does one `tm__atomic_load(&g_table)` with no lock, bounds-checks the id, and loops the handlers.
- **Memory ordering:** the execute load is an acquire, and it pairs with the writer's seq-cst store. A reader that sees the new table pointer also sees every write that built the table.

## 3. Spans

```
struct TmSpanEvents { start: int32, stop: int32, exception: int32 }
struct TmSpan { start: int32, stop: int32, exception: int32, start_mono_ns: int64 }  // ids by value: cannot dangle

fn tm_span_events(prefix: string) -> TmSpanEvents      // registers prefix.start / .stop / .exception
fn tm_span_start(ev: TmSpanEvents*, meta: TmKV*, n_meta: int32) -> TmSpan
fn tm_span_stop(s: TmSpan*, meas: TmKV*, n_meas: int32, meta: TmKV*, n_meta: int32)
fn tm_span_exception(s: TmSpan*, kind: string, reason: string,
                     meas: TmKV*, n_meas: int32, meta: TmKV*, n_meta: int32)
```

- **`.start`** measurements are `system_time` (wall ns) and `monotonic_time`. Metadata is the caller's meta.
- **`.stop`** measurements are `duration_ns` (monotonic, computed by tm), followed by the caller's meas. These are built in a stack array of `TM_SPAN_MAX_MEAS = 16`. Anything beyond that is truncated.
- **`.exception`**: same `duration_ns`, plus meta `kind` and `reason` appended to the caller's meta, also capped at 16.
- `.start` carries `system_time`; `.stop` and `.exception` do not (same as `:telemetry`).
- When the target event has no handlers, `tm_span_stop` and `tm_span_exception` return before building the 16-entry array.
- Start and stop still read the monotonic clock (about 20 ns per vDSO call). Callers that care can check `tm_has_handlers` first.

## 4. Clock (public, until std/time exists)

```
fn tm_monotonic_ns() -> int64    // CLOCK_MONOTONIC
fn tm_system_time_ns() -> int64  // CLOCK_REALTIME
```

## 5. Metrics (reducers attached as handlers): v0.2

```
const TM_COUNTER = 0; TM_SUM = 1; TM_LAST_VALUE = 2; TM_HISTOGRAM = 3;

struct TmMetricDef {
  kind: int32, event: string,          // exact event name (attached via prefix = event name)
  measurement: string,                 // ignored for COUNTER
  tags: string*, n_tags: int32,        // metadata keys; values become the series key
  buckets: int64*, n_buckets: int32,   // HISTOGRAM only: ascending upper bounds; +Inf implied
  max_series: int32,                   // cardinality cap; overflow → TM_ERR_FULL counted in `dropped`
}
fn tm_metric_new(def: TmMetricDef*) -> TmMetric*   // attaches itself; copies def
fn tm_metric_free(m: TmMetric*)                    // detaches; call after threads stop using it

struct TmSeries { tags: TmKV*, n_tags: int32, count: int64, sum: int64, last: int64,
                  buckets: int64*, n_buckets: int32 }      // snapshot copy, caller frees via tm_series_free
fn tm_metric_series_count(m: TmMetric*) -> int32
fn tm_metric_snapshot(m: TmMetric*, i: int32, out: TmSeries*) -> int32
fn tm_metric_dropped(m: TmMetric*) -> int64
```

- **Values are int64** (ns, bytes, counts). F64 measurements are rounded and bool counts as 0/1. A string measurement is skipped.
- **Series storage** is a fixed open-addressing table of `max_series` slots, keyed by an FNV-1a hash of the tag values (strings copied on insert).
  - New series are inserted under the metric's mutex.
  - Existing series update lock-free: `tm__atomic_add` for count, sum and buckets, and `tm__atomic_store` for last.
- **Snapshots are not atomic across fields**: count and sum may be one event apart. That is acceptable for monitoring.
- Missing tag key → the tag value is `""`.

## 6. Layout

```
src/telemetry.mettle             public facade (re-exports); import "telemetry/telemetry"
src/value.mettle                 TmValue, TmKV, constructors, tm_kv_find
src/core.mettle                  table build, prefix match, init/shutdown, tm_event, attach/detach, execute
src/span.mettle                  spans
src/metrics.mettle               reducers, series table, snapshots (v0.2)
src/sys/atomic.mettle            tm__load_ptr / tm__store_ptr_sc (@naked x86-64)
src/sys/clock.mettle             clock_gettime extern
tests/                           see below
mbuild.mettle                    mbuild manifest at root: mb_package("telemetry", ...), lib only
```

## 7. Acceptance tests
Every test runs in both debug and `--release`, because Mettle has release-only miscompiles.

1. `tm_event` is idempotent; ids are dense from 0. `tm_init` is idempotent.
2. Prefix match is per segment: `a.b` matches `a.b` and `a.b.c` but not `a.bc`; `""` matches everything.
3. An event registered after an attach still receives the handler.
4. Attaching a duplicate `handler_id` returns `TM_ERR_EXISTS`. A detach removes the handler from every event. Detaching an unknown id returns `TM_ERR_NOT_FOUND`.
5. A handler matched by two prefixes runs once per event. Handlers run in attach order.
6. Executing with no handlers, or with an invalid id, makes no calls and does not crash.
7. Spans:
   - The `.stop` duration is ≥ 0 and ≥ a measured sleep.
   - Only `.start` carries `system_time`.
   - `.exception` carries `kind` and `reason`.
   - The 16-entry cap holds.
   - A span copied by value still works.
8. **Stress:**
   - Setup: 8 threads call `tm_execute` 1M times while 1 thread attaches and detaches a second handler in a loop.
   - There are no crashes.
   - A handler attached throughout, with an atomic counter, counts exactly 8M.
9. **Benchmark:** `tm_execute` with no handlers costs under about 5 ns per call in `--release`. Also report `tm_has_handlers`.
10. **Metrics (v0.2):**
   - Counter, sum, last and histogram bucket counts are exact under 8 threads.
   - The `max_series` overflow increments `dropped`.
   - Tags with the same values map to one series.
11. A handler calling `tm_attach` from inside a handler does not deadlock.

## 8. Resolved in review (raize_web, 2026-11-05)
1. `tm_init` is required and idempotent. Both `rz_run` and the app may call it.
2. INSTR needs only events and spans, so metrics ship in v0.2.
3. Linux x86-64 only is fine; raize's poll loop already is.
4. `system_time` goes on `.start` only.
5. `cfg` stays valid until shutdown; execute uses acquire/seq-cst ordering; `tm_has_handlers` is public; `TmSpan` holds ids by value; the manifest is `mbuild.mettle`; tests run in release and include a benchmark.
