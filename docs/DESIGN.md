# Design

## Why this exists

Gemini CLI is deprecated for personal use, so Paperclip's built-in `gemini_local`
adapter has no viable lane for personal accounts. `agy` (Antigravity CLI) is the
replacement CLI. This package adds an `agy_local` adapter through Paperclip's
external-adapter extension point, so nothing in Paperclip needs to be forked.

## Where this plugs into Paperclip

Paperclip loads external adapters through
`@paperclipai/server/dist/adapters/plugin-loader.js`:

1. `POST /api/adapters/install` (instance-admin only) resolves the package, either
   `npm install`ing it into `~/.paperclip/adapter-plugins/` or resolving a local path.
2. `resolvePackageEntryPoint()` reads `exports["."].import` (falling back to `main`).
3. `validateAdapterModule()` imports that entry and requires a **`createServerAdapter()`**
   export returning a `ServerAdapterModule` with a non-empty `type`. It then runs
   `validateAdapterLoginCapability()`, which fails closed on a malformed capability.
4. If `exports["./ui-parser"]` exists, its **source** is read and cached, gated on
   `package.json` → `paperclip.adapterUiParser` having major version `1`.
5. The registration is persisted to `~/.paperclip/adapter-plugins.json` and reloaded on
   startup by `buildExternalAdapters()`.

`BUILTIN_ADAPTER_TYPES` in the server states that external plugins must not *replace*
built-in types but may add new ones. `agy_local` is a new type, so it does not collide.

`scripts/verify-loader.mjs` reimplements steps 2–4 exactly, so a contract break is
caught locally rather than at install time.

## agy stream-json protocol

Captured from `agy --output-format stream-json` on agy 1.1.28. One JSON object per
line, discriminated by a top-level `event`.

### `init`

```json
{"event":"init","conversation_id":"1d40…","init":{"cwd":"…","tools":["view_file",…],"permission_mode":"always-proceed"}}
```

`conversation_id` is the resume handle. Note `init.cwd` reports the *process* cwd,
which is not necessarily where agy will write — see "Workspace binding" below.

### `step_update`

```json
{"event":"step_update","step_update":{
  "conversation_id":"1d40…","step_index":1,
  "state":"ACTIVE"|"DONE",
  "step_type":"user_input"|"agent_response"|"tool",
  "text_delta":"…","tool_name":"write_to_file",
  "tool_info":{"name":"…","parameters":{…},"output":"…"},
  "duration_seconds":2.15,
  "usage":{"input_tokens":5286,"output_tokens":90,"thinking_tokens":86,"cache_read_tokens":8128,"total_tokens":5376}}}
```

- `text_delta` is a **true incremental chunk**. Concatenating deltas in order
  reconstructs the message, so the UI parser marks assistant entries `delta: true`.
- A tool step appears **twice**, ACTIVE then DONE, sharing one `step_index`. Both
  parsers key on `step_index` so the pair collapses into one invocation rather than
  rendering twice.
- `step_index` is not dense — observed runs skip indices.

### `result`

```json
{"event":"result","result":{"conversation_id":"1d40…","status":"SUCCESS","response":"…","duration_seconds":7.6,"num_turns":1,"usage":{…}}}
```

`result.usage` is the **sum over the whole invocation**, not the last step. In a
two-tool probe the final step reported 6,613 input tokens while the result event
reported 18,259. Hence `usageBasis: "per_run"`; reporting `session_cumulative` would
make Paperclip delta against the previous run and undercount.

Only `SUCCESS` has been observed. Non-success handling is written defensively: the
parser reads several plausible error field names and, failing that, synthesizes
`agy finished with status <STATUS>`.

## Workspace binding — the critical behaviour

**agy does not use its process cwd as the workspace.** A probe run with cwd set to a
temp project directory wrote to `~/.gemini/antigravity-cli/scratch/probe.txt` instead.
Passing `--add-dir <cwd>` corrected it; the same prompt then wrote to the intended
directory.

This is the highest-severity failure mode for this adapter, because it is silent:
the run succeeds, the model reports success, and the workspace diff is empty. Agent
work would disappear with nothing to review.

Mitigations:

- `buildAgyArgs()` passes `--add-dir` unconditionally, not behind a config flag.
- A unit test asserts `--add-dir` is present and carries the resolved cwd.
- `scripts/verify-e2e.mjs` asserts a real file lands in a real workspace.
- The behaviour is documented in the README and `agentConfigurationDoc`.

## Session model

`sessionParams` is `{ conversationId, cwd, workspaceId?, repoUrl?, repoRef? }`.

`cwd` is part of session identity on purpose. An agy conversation is bound to the
directory it was created in; resuming it elsewhere hands the model a transcript
describing files that are not present. `execute()` therefore resumes only when the
stored cwd resolves equal to the current execution cwd, and logs a `[paperclip]` line
explaining the refusal otherwise.

`sessionCodec.deserialize()` also accepts a bare string and a `{sessionId}` object, so
rows written by Paperclip's legacy single-session view still decode.

`sessionManagement` declares `nativeContextManagement: "unknown"`, matching what
`gemini_local` declares. agy resumes conversations but documents no automatic
compaction, so Paperclip keeps its threshold-based session rotation active
(200 runs / 2M raw input tokens / 72 hours).

### Recovery

A resume that fails with a "conversation not found"-shaped error triggers exactly one
retry with a fresh conversation. On that retry the adapter does **not** fall back to
the stored conversation id, since it is known-stale. If no conversation id can be
resolved on a failed run, `clearSession: true` tells Paperclip to drop the dead handle
so the next heartbeat starts clean.

## Timeout interaction

Two timers exist: Paperclip's `timeoutSec`, which kills the process, and agy's own
`--print-timeout`. If Paperclip's fired first, agy would die mid-stream before emitting
a result event, losing the conversation id and orphaning the session.

`resolveAgyPrintTimeoutSec()` sets agy's timeout to `timeoutSec` minus a 5% margin
(minimum 10s, floor 30s), so agy always exits first and reports its own state.

## Failure classification

`execute()` maps failures to distinct error codes so Paperclip can act on them:

| Condition | `errorCode` | `errorFamily` |
|---|---|---|
| Not signed in / 401 | `agy_auth_required` | — |
| Quota / rate limit / 429 | `agy_quota_exhausted` | `provider_quota` |
| Connection reset, 5xx, DNS | `agy_network_unavailable` | `transient_upstream` |
| Transport-level failure | forwarded from `proc.errorCode` | — |

`errorFamily` is what drives Paperclip's retry and escalation behaviour, so quota and
transient-network failures are tagged rather than surfacing as opaque crashes.

A run is treated as failed when **any** of: non-zero exit, a parsed non-`SUCCESS`
status, or **no result event at all**. That last case matters — a truncated stream with
a zero exit code would otherwise read as success.

## Security boundaries

- `applyConfiguredEnv()` refuses `PAPERCLIP_API_KEY` from adapter config
  (`isForbiddenConfigEnvKey`) and refuses to shadow any `PAPERCLIP_*` key the runtime
  already assigned. The harness-minted run token stays the only source of Paperclip API
  identity, so operator config cannot hijack the control plane.
- The prompt is always the final CLI argument, so `onMeta` can replace exactly that one
  element with `<prompt N chars>` and keep prompt bodies out of duplicated log fields.
- Env is logged through `buildInvocationEnvForLogs()`, which applies Paperclip's
  redaction.
- `ui-parser.cjs` runs in the host's locked-down worker (no network, no storage, no
  module loader). It is plain self-contained CJS with no `require`; a test asserts both
  properties and evaluates it in a bare `vm` context to prove it.

## Deliberate scope limits

- **Local execution focus.** `execute()` routes through
  `runAdapterExecutionTargetProcess`, so SSH and sandbox targets are structurally
  supported and session identity is recorded per target. But only the local target has
  been verified, and `testEnvironment()` reports an explicit warning rather than a
  false pass when a remote target is configured.
- **No ACP lane.** agy exposes no ACP server, so unlike `claude_local` and
  `gemini_local` there is no `acp` descriptor.
- **No skill sync.** `listSkills`/`syncSkills` are unimplemented; agy's skill directory
  layout has not been verified. `instructionsFilePath` covers agent instructions in the
  meantime. This is the most likely next feature.
- **No login capability.** agy's login is an interactive browser flow with no
  scriptable `setup-token`-style equivalent, so no `loginCapability` is declared.
  Authentication is a one-time manual `agy` run, which `testEnvironment()` verifies by
  listing models.

## Verification performed

Against agy 1.1.28 and Paperclip 2026.831.1, macOS arm64:

| Check | Result |
|---|---|
| `validateAdapterModule()` replay | pass — `type = agy_local` |
| ui-parser contract extraction | pass — 6,096 bytes, contract 1.0.0 |
| Live model discovery | pass — 15 models |
| `testEnvironment()` | pass — agy found, authenticated |
| Workspace binding (real file write) | pass — file landed in workspace |
| Session resume (real recall) | pass — same id, model recalled prior turn |
| Stale-session rejection | pass — refused resume, warned, new id |
| Unit tests | 29/29 pass |

Not yet done: registration in a running Paperclip instance (needs instance-admin), and
any Linux or Windows testing.
