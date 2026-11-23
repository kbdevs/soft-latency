---
name: soft-latency
description: Cut avoidable non-inference ("soft") latency across an AI chat or streaming application — server setup, post-answer tail, router dispatch, tool wrappers, database fences, and web/mobile clients — while preserving safety, auth, billing, and durability invariants. Audits start read-only.
---

# Cut soft latency

Remove the latency your own code adds around model inference, external vendor
APIs, and network round-trips. Do not try to out-optimize fixed physics or model
decode time; eliminate every unnecessary wait, serial round-trip, cache miss,
and transport stall you control. Preserve auth, tenant isolation, billing,
safety screening, durability, and parity invariants.

## Start read-only

For an audit, read source code, traces, telemetry schemas, and permitted
production timing aggregates first. Report the full latency breakdown and
ranked soft-latency targets before editing any file. Do not edit code, change
infrastructure, or deploy until the user authorizes that work.

## Split the clock before touching code

Total end-to-end latency hides where time actually goes. Measure and separate
two user-visible clocks:

- **Time to first visible answer word** — from user send to the first answer
  token rendered on screen (not raw GPU TTFT, which misses client, setup,
  routing, and pre-answer tool hops).
- **Time to answer complete** — from user send to stream finish and persistence.

Decompose every request into chronological, additive stages:

1. **Client & network** — auth lookup, route navigation, cold connection pool,
   TLS/RTT, stream compression buffering, and client token-smoothing tail.
2. **Server setup** — request parse, auth, rate-limit/quota checks, history and
   memory loads, attachment verification, and session/lock claims.
3. **Router & admission** — capacity admission, health/circuit reads, provider
   selection, media URL signing, and stream prefix holds.
4. **Tool wrappers & multi-hop dispatch** — tool-call ledger writes, quota
   reads, DNS/robots/page-fetch fallbacks, timeout ceilings, and hop-2+ setup.
5. **Confirmation & persistence fences** — transactional metering, freshness
   checks, retirement/erasure guards, and object-store sidecar reads.
6. **Post-answer tail** — stream flush, coordinator RPCs, final message save,
   and lock release.

Segment by turn archetype (plain text, with attachments, image generation, web
search, URL fetch, multi-hop tools). Fleet averages hide 5x to 15x tail waits
on tool and media paths. Separate **hard floors** (model prefill/decode,
external API execution, wire RTT) from **soft overhead** (your code).

## Attack the six soft-latency layers

Prioritize critical-path waits that run on every request or dominate slow
archetypes. Parallelize independent streams of work when changes touch disjoint
subsystems, and keep every change small and verifiable.

### 1. Server setup (pre-first-token)

- Start independent reads (history, memory, feature flags, tool controls,
  attachment probes) as early as request parsing allows and await them together.
- Use a dedicated read-only connection for side lookups instead of serializing
  them behind the turn's write transaction.
- Replace check-then-insert pre-reads with optimistic insert plus duplicate-key
  fallback when the fast path is a new record.
- Skip tombstone or existence checks that are structurally impossible for freshly
  server-minted IDs.
- Skip optional context loads (such as pending integration or plugin state) when
  the caller has none connected.
- Memoize hot config and feature-flag reads in-process with a short TTL and
  in-flight request coalescing. **Bound every cross-request shared promise**
  with each joiner's own timeout and keep the starter's read alive across client
  disconnects so an aborted starter can never hang joiners.

### 2. Post-answer tail, stateful coordinators & DB transactions

- Coalesce sequential coordinator or Durable Object RPCs into batched writes
  and fold trailing frames (such as saved-message metadata or suggestions) into
  the final `settle` call.
- Skip no-op empty flushes and overlap stream flush with final DB persistence
  when ordering permits.
- Fold redundant post-write metadata updates into the primary admission write.
- Cache per-connection session configuration resets (for example transaction
  timeout resets) on pooled connections so repeat transactions do not pay extra
  SQL round-trips.
- Keep async post-turn accounting, billing settlement, or analytics off the
  turn's serial database connection.

### 3. Router, admission & multi-hop dispatch

- Skip synchronous remote admission checks when routing to local or home
  capacity that already tracks its own admission.
- Serve circuit-breaker, node-health, and routing-fallback state
  stale-while-revalidate with single-flight background refresh instead of
  blocking request dispatch.
- Memoize parsed routing weights and static serialized tool schemas across hops.
- Warm and share media access grants across nodes and hops rather than
  re-signing on every hop.
- Replace blanket leading-character stream holds (used to detect markers or
  prefixes) with exact incremental prefix matchers so non-matching tokens flush
  immediately.
- Eliminate avoidable extra LLM recovery hops by injecting explicit turn-context
  notices before forcing a final no-tool answer pass.

### 4. Search, URL fetch & tool wrappers

- Do not pay synchronous database ledger round-trips for pure read-only tools.
  For stateful tools that need a ledger, prefer insert-first over read-then-write.
- Decouple tool-end telemetry and UI updates from blocking settlement drains,
  and parallelize quota checks across concurrent tool calls.
- Cache DNS lookups (opt-in short TTL), `robots.txt`, fast-host misses, and
  queryless public pages in the edge PoP cache.
- Tune fast-path to fallback handoffs and separate general web-page timeout
  ceilings from intentionally slower document (PDF) caps.
- Fix in-flight deduplication bugs (such as shared pending-map keys) and reuse a
  single pooled DB connection inside search tools.

### 5. Attachment & media confirmation fences

- Collapse duplicate lock, meter, freshness, and retirement SQL round-trips
  inside transactional confirmation fences without weakening any lock order or
  erasure guarantee.
- Skip object-store extraction sidecar reads when the inline text or metadata
  is already complete.
- Overlap read-only source-header probes with safety screening while gating any
  side effect on the screening verdict.

### 6. Web, iOS & Android clients

- Unify REST and streaming/upload HTTP connection pools so warm connections are
  reused, and flush or prewarm stale mobile pools on app foreground (off the
  send clock).
- Send explicit `Accept-Encoding: identity` on SSE and streaming endpoints when
  the HTTP client otherwise applies transparent gzip buffering.
- Emit idle/complete immediately when token smoothers drain instead of waiting
  an extra animation or frame timer tick.
- Read valid cached auth tokens synchronously on send, and dispatch the chat
  request before triggering client-side route navigation on new threads.

## Guardrails

- **Never weaken correctness for speed.** Keep auth checks, tenant scope,
  quota/billing gates, safety screening, retirement/erasure fences, and
  durability ordering intact.
- **No unbounded shared promises.** Any module-level in-flight map or promise
  shared across requests must enforce a per-caller timeout and survive starter
  cancellation.
- **No silent regressions on fallback paths.** Test error, duplicate-key,
  cache-miss, timeout, and abort paths alongside every fast path.
- **Respect protected gates.** Never deploy to production, bypass schema or
  policy review gates, or fake visual/client verification.

## Prove the change

Verify every modified layer with the narrowest complete set of checks:

- Typecheck, lint, formatting, and repository static invariants (shared-read
  bounds, boundary casts, file limits, contract/parity checks).
- Unit tests covering fast paths, fallbacks, races, and error handling.
- End-to-end or worker runtime pipeline tests for streaming, tool execution,
  and persistence ordering.
- Client unit/UI tests for every touched platform; state explicitly any
  platform build or test suite that could not run locally.

## Deliver

Report:

1. **Baseline breakdown** — hard floors vs. soft overhead, segmented by turn
   archetype, with exact sample window and source.
2. **Changes by layer** — the specific wait removed, the files touched, and the
   invariants preserved.
3. **Before vs. after table** — clearly labeling every `After` number as
   **observed production**, **local benchmark**, or **modeled projection** from
   eliminated RTTs/waits. Never present a modeled projection as a live
   production measurement, and never sum non-additive per-path savings into a
   fake total.
4. **Verification ledger** — exact commands run, pass counts, and anything left
   for CI or post-deploy production telemetry.

If asked for a public chart or post, use direct labels, a zero baseline, and no
internal names, ticket IDs, credentials, or private links. Keep methodology and
caveats in the post or README text rather than on the image. Publishing to a
public repo or social channel requires the user's explicit go-ahead.
