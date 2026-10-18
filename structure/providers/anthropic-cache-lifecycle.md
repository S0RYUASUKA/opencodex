# Anthropic Cache Lifecycle

`src/anthropic/cache-renewal.ts` owns confirmed five-minute prompt prefixes and their renewal
clock. `src/anthropic/cache-renewal-runtime.ts` binds that owner to Responses transport, Codex
runtime state and usage logging. This does not change native Codex or native Messages passthrough.

## Running-turn renewal

The optional top-level `anthropicCacheRenewal` boolean defaults off. Only explicit `true`
enables background sends; existing `cacheRetention` choices remain authoritative. One-hour,
disabled, externally controlled or incompatible cache requests do not opt into renewal.
Configured root, identity or pool spend ceilings disable renewal, including a ceiling added after
confirmation. Detached renewals do not yet reserve their original workflow scopes. Empty spend
scopes and retention-only accounting stay observational; normal sends retain existing admission.

`src/codex/request-provenance.ts` derives the request's own thread and turn from HTTP or
per-frame WebSocket metadata. Parent conversation identity is not an own-thread fallback.
Main and spawned turns follow the same policy; decision, memory and vision helpers do not.
`src/codex/cache-runtime-state.ts` reads the existing local IPC state. When IPC cannot identify
a headless child, `src/codex/rollout-cache-runtime.ts` reads that child's native lifecycle and
`src/codex/thread-writer-lock.ts` verifies the Windows writer lock. Other platforms without
usable IPC fail closed; the default endpoint currently supports Windows only. Known large compacted/context records do not imply completion;
unknown records, missing locks, changed turns and explicit terminal events stop renewal.

The clock starts at physical send, after complete valid upstream cache usage confirms a
prefix. At 270 seconds, the manager checks the same active turn and sends the same full
prefix with zero output tokens. It changes no tool definitions, historical messages or
credential. The request pins destination, model and credential; OAuth ownership is checked
again before dispatch. This is a billable cache-read request, not a free heartbeat.

Natural usage observations replace the confirmed prefix and reset its clock. During a long
generation the previously confirmed prefix remains eligible until the new request's early
usage arrives. `src/lib/anthropic-cache-observation.ts` carries the observer through response
wrappers, including `src/web-search/progress-stream.ts`; it does not consume a second stream.
Incompatible requests clear old eligibility. Completion, cancellation, unknown runtime state,
expired confirmation and disabled config stop further sends.

Retries are bounded by attempts and remaining confirmed lifetime. Ambiguous post-send failure
is not a confirmed renewal or zero cost. Renewal never rotates accounts, publishes an answer,
executes a tool or appends anything to client history. Diagnostic records correlate request
ids with allowlisted thread/turn/role, phase, counters and fixed network reasons; they contain
no prompt, image, credential or raw exception. `src/server/request-origin.ts` separately
correlates natural request logs, keeping missing/conflicting provenance unknown.
Terminal continuations select Anthropic SSE parsing from the actual response content type even
for a non-stream client. Both initial and continuation 413 paths refuse unchanged image resends.

## Stable images and overflow recovery

`src/adapters/anthropic-image-normalize.ts` chooses encoding from the image itself, not its age,
aggregate pressure or retry number. A repeated image is deterministic even after cache eviction.
Single-image validation and decoding protections remain in place. The actual Anthropic adapter
uses the non-mutating assertion in `src/adapters/anthropic-image-guard.ts`: 26 MiB aggregate
base64, the existing image-count/dimension limits, and 32,000,000 bytes for the complete
serialized request, including system, tools and text. An upstream 413 is not retried with
historical pictures degraded. The legacy mutating helper is not used by this adapter.

`src/server/responses/anthropic-image-context.ts` partitions history only at complete tool
call/result boundaries. On local overflow it summarizes the largest fitting old prefix and
retains the newest indivisible unit, including the latest image. Recovery repeats only while
the same admission, cancellation and finite send budget allow it. An individually oversized
new unit or unverifiable owner fails explicitly; no finite API limit can admit every image.

`src/server/responses/image-context-recovery.ts` dispatches the summary inside the original
Responses owner. `src/server/responses/response-effects.ts` preserves Anthropic tools, system
and normal tool choice. `src/responses/compaction-summary.ts` suppresses client tool events
and rejects invalid summaries before checkpoint publication, including EOF without a normal stop
reason. Successful summaries have no failure error code. Sidecar execution is bypassed.
Hosted server tools are the exception: `tool_choice: none` prevents upstream execution, at
the documented cost of invalidating the Messages cache when choice changes.

Successful summaries use the existing bounded response-state store, bound to principal, own
thread and serving owner. A digest identifies the exact timestamp-free historical prefix
without retaining another full image history. Replay reuses a matching checkpoint; a failed
store write refuses further summary charges. This is a proxy projection, not a rewrite of the
native Codex transcript. After necessary compaction, a new message prefix is expected.

## Verification boundaries

The focused cache lifecycle entry tests use generated images, fake upstreams, blocked external
connections and a deterministic clock. They exercise HTTP/WS, compact v1/v2, image overflow,
checkpoint reuse, default-off behavior and usage correlation. Runtime record tests use synthetic
large native records. These checks establish the wire and lifecycle contracts, not live provider
acceptance or a guarantee that every future request hits a cache.
