# token-bucket-lib

One continuous-refill token bucket per key, over an injected clock. Pure
arithmetic: no host, no database, no queue, no `host.time()`.

Extracted from discofetch `api/supervisor.lua` (the "meter" section,
`BUCKETS` / `BUCKETS_MAX` / `buckets_count` / `buckets_shed` / `rate_allow`).
The arithmetic, the branch order and the refusal behaviour are unchanged.
What was module-level mutable state there is instance state here, which is
the only structural change.

## Surface

```
M.new{ now = <fn () -> unix SECONDS>, max_keys = <n> } -> instance

instance:allow(key, per_hour, burst) -> true
                                     -> false, <whole seconds until a token>
instance:count()      -> live keys, i.e. keys created since the last shed
instance:shed_count() -> how many times the table has been dropped whole
instance.max_keys     -> the cap count() is racing

M.DEFAULT_MAX_KEYS = 100000
```

`max_keys` is a field rather than a call because the two are printed
together: a count published without the cap it is racing is not worth
printing. `GET /v1/admin/stats` reads `count()`, `max_keys` and
`shed_count()` in one card.

**ONE instance per deployment.** The limiter and the stats handler are handed
the same one. Two instances split the counters and the card under-reports
both.

## Injected deps

| dep | required | contract |
| --- | --- | --- |
| `now` | yes | zero-arg function returning whole unix **SECONDS** |
| `max_keys` | no | number of keys >= 1; defaults to `M.DEFAULT_MAX_KEYS` |

Both are checked at `new()` with a sentence naming the dep, so a wrong dep is
a named failure at construction rather than a nil call on the serving path an
hour later.

The clock's unit is load-bearing, not decorative. The refill term
(`(t - b.at) * per_hour / 3600`) and the `ceil` that computes the wait are
second-resolution by construction; a millisecond clock changes every rate by
1000x. Nothing in this library may call `host.time()` — supervisor owns the
one place that knows `host:time()` answers in milliseconds.

## Usage

```lua
local bucket = token_bucket.new{ now = now }          -- now() -> unix seconds

local allowed, wait = bucket:allow('dns:' .. label, tier.dns_per_hour, tier.dns_burst)
if not allowed then
  return 429, { error = { code = 'rate_limited', message = 'too many; retry in ' .. wait .. 's' } }
end
```

An allowed call returns `true` and **nothing else** — absence, not a zero a
caller could hand to a `retry-after` header. The wait is only present on a
refusal, is whole seconds, and is floored at 1 so nobody is told to retry in
0 seconds.

Key derivation, the policy numbers, the 429 body and the `retry-after` /
`cache-control` headers all belong to the caller; see below.

## Why this is memory, and why the shed counter exists

The buckets and the counters are memory deliberately, holding two invariants:
root never hibernates, and nothing here touches the sql connector. A rate
window that survives a deploy is not worth a database write per request, and
a counter that resets on restart says so in its answer. That is the standing
answer to the first proposal to persist these tables, and the reason this
library has no db dep to give one.

At `max_keys` a **new** key drops the whole table rather than being refused:
the cost is one fresh burst for everyone, the alternative is stale keys
squatting while new callers are turned away. It fails open on purpose — an
LRU here would be a refusal nobody asked for, so do not optimise it into one.

A shed is invisible from inside a request; everyone simply gets a fresh
burst. From outside it looks like the rate limiter briefly not being there,
which is the one symptom nobody would think to attribute to a memory cap.
`shed_count()` is the difference between that being a question with an answer
and a question nobody can ask.

`max_keys` is a **memory backstop, far above honest use**. It is not a
traffic dial — `per_hour` and `burst` are the dials. Tuning it down to shape
traffic sheds every caller's window at once instead.

## Dependency edges

This library is the floor: **it depends on nothing.** No other library in the
set is imported, and none may be.

Owned elsewhere, deliberately not copied here (a copy is how two libraries
drift):

- `rate_headers(wait)`, the `rate_limited` refusal body and the policy
  numbers (`redeem`, `authfail`) — **token-rate-limit-lib**, which takes one
  instance of this library as its only dep and does the key derivation and
  unit conversion at its call sites.
- `TIERS` / `tier_of`, which supply `per_hour` and `burst` —
  **discofetch-accounts-lib**.
- `now` — supervisor's composition root, injected here.
- The rooms store, which shares the literal 100000 and nothing else:
  **discofetch-fetchpoint-lib**. Rooms refuse the new entry at their cap;
  buckets shed and fail open. Matching numbers, opposite policies — not a
  shared bounded-map abstraction.

## Known, carried over

Behaviour I believe is wrong or fragile and did **not** change, because the
extraction preserves behaviour:

- **`per_hour` and `burst` are not validated.** `per_hour = 0` makes `rate`
  zero, and the refusal path divides by it: under drt 0.5.0 that yields `inf`
  rather than an error, so the caller would advertise `retry-after: inf`.
  Every call site passes a positive constant today.
- **`burst` is per call, not per key.** It is both the initial fill and the
  ceiling in `math.min`, so passing a different `burst` for the same key
  changes that key's ceiling from then on. Every call site passes one tier's
  number consistently today.
- **A clock that goes backwards removes tokens.** `(t - b.at)` is signed and
  unclamped.
- **`math.max(1, ...)` is belt-and-braces.** The branch is only reached with
  `tokens < 1`, so `ceil` of a positive number is already >= 1; the floor can
  only bite on an infinite or NaN rate.
- **The advertised wait is a float divide then a `ceil`**, so a rate that is
  not a binary fraction can round up by one second. Always conservative —
  never early — so it is left alone.
- **Nothing expires.** A key occupies a slot until the next shed; the shed is
  the only reclamation. That is the design, stated here so it is not
  mistaken for an oversight.

## Deliberately left out

No persistence, no LRU or per-key expiry, no headers, no HTTP, no key
derivation, no policy table, no traffic counting, and no logging. Every one
of those belongs to a named owner above.

## Consumption waits on `require`

Guests in DRT have no `require` and no `dofile` today; the load-time modules
slice is designed but unshipped. This is a real module — the file ends in
`return M` — and nothing imports it yet. That is expected.

Until then, tests wrap the module in an IIFE and concatenate the cases:

```
sh test/run.sh          # DRT=/path/to/drt to override; last line is PASS
```

When `require` lands, `test/run.sh` becomes two `require` lines.
