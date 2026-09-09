# Upstream proposal — draft outreach to the Paperclip maintainers

**Status: DRAFT — not sent.** Sending requires sign-off from the repository owner.

Suggested channel: a GitHub issue on [`paperclipai/paperclip`](https://github.com/paperclipai/paperclip/issues),
which keeps the technical discussion public and reviewable. Email is the fallback if
the maintainers prefer it.

---

**Subject:** Offering `agy_local` — a maintained Antigravity CLI adapter — for the Paperclip org

Hi Paperclip maintainers,

Gemini CLI is now deprecated for personal use, which leaves the built-in `gemini_local`
adapter without a usable lane for personal accounts. We needed a replacement, so we
built an adapter for **agy**, the Antigravity CLI, and we would like to offer it to the
Paperclip organisation rather than maintain it as a private fork of the ecosystem.

Repository: https://github.com/evgemar/agy-paperclip-adapter (MIT)

**What it is**

A standard external adapter package — `createServerAdapter()` returning a
`ServerAdapterModule` with type `agy_local`, loaded through your existing
`plugin-loader` path. No Paperclip patch or fork is required, and it adds a new type
rather than overriding a built-in one.

It implements the full practical surface: `execute`, `testEnvironment`, a session codec
with conversation resume, live model discovery from `agy models`, a declarative config
schema, `getRuntimeCommandSpec`, `agentConfigurationDoc`, and a `./ui-parser` export at
contract 1.0.0 for live log rendering.

**The finding we think is worth your attention**

agy does not treat its process working directory as the workspace. Launched with `cwd`
set to a project directory, it wrote its output into `~/.gemini/antigravity-cli/scratch/`
instead. Passing `--add-dir <cwd>` fixes it.

This matters because the failure is silent: the run exits zero, the model reports
success, and the workspace diff is empty — agent work disappears with nothing to review.
Our adapter passes `--add-dir` unconditionally and has a test asserting it cannot
regress. If you adopt or reimplement this adapter, that flag is the one thing not to
drop.

**Verification**

Against agy 1.1.28 and Paperclip 2026.831.1 on macOS (arm64):

- Loader contract replayed locally (`validateAdapterModule`, UI-parser extraction) — pass
- Live model discovery (15 models) and `testEnvironment` — pass
- Real run writing into a real workspace — file lands in the workspace
- Session resume — same conversation id, model recalled the prior turn
- Stale conversation id (cwd mismatch) — refused, operator warned, fresh conversation started
- 29 unit tests over captured `stream-json` fixtures
- Fresh `git clone` → `npm install` → `npm run build` → `npm test` — pass

Linux and Windows are untested; we would value help there.

**Known gaps, stated plainly**

- No skill sync (`listSkills`/`syncSkills`) — agy's skill directory layout is unverified.
  `instructionsFilePath` covers agent instructions today. This is the most likely next
  feature and we would follow your preferred pattern.
- No ACP lane — agy exposes no ACP server.
- No `loginCapability` — agy's login is an interactive browser flow with no scriptable
  `setup-token` equivalent. Authentication is a one-time manual `agy` run, which
  `testEnvironment` verifies by listing models.
- Only the local execution target is verified. Remote targets route through
  `runAdapterExecutionTargetProcess` and are structurally supported, but
  `testEnvironment` reports an explicit warning rather than a false pass.

**What we are asking**

We would like to transfer the repository into the Paperclip organisation, or have it
listed as a recommended community adapter — whichever fits your governance. We are happy
to:

- adopt your package naming (`@paperclipai/adapter-agy-local`) and release cadence
- move it into `packages/adapters/` as an in-tree adapter if you prefer that shape
- keep maintaining it under the Paperclip org, including the gaps above

If the answer is a reimplementation in-tree rather than adoption, that is a fine outcome
too — please take the `--add-dir` finding and the protocol notes in
[`docs/DESIGN.md`](DESIGN.md), which document the agy `stream-json` event shapes and the
per-run usage semantics.

Happy to open a PR in whatever form is most useful.

Thanks,
Agency Agents
