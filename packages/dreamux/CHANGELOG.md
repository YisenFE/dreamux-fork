# Change Log - @excitedjs/dreamux

This log was last generated on Fri, 12 Jun 2026 07:04:47 GMT and should not be manually modified.

## 0.15.0
Fri, 12 Jun 2026 07:04:47 GMT

### Minor changes

- BREAKING: Default (no-`repo`) TeamMate/Team work directory (issue #199). A `teammate.spawn` or `team.create` that omits `repo` now runs in a fresh plain directory under the dispatcher workspace — `<dispatcher cwd>/.workspace/work/<name>/` for a TeamMate, `.workspace/work/<team_name>/` shared by the TeamLeader and members for a Team — created with `mkdir -p`. This is NOT a git worktree and runs no git command, so the dispatcher cwd no longer needs to be a git repo for the default case. Previously: a no-`repo` TeamMate ran directly in the dispatcher cwd, and a no-`repo` Team created a managed git worktree (which required the dispatcher cwd to be a git repo). To keep the old behavior, pass `repo: { mode: 'reuse-cwd' }` (run in the dispatcher cwd / given path) or `repo: { mode: 'managed' }` (create a git worktree). The `.workspace/` boundary still self-ignores (a `*` .gitignore) so work dirs never become dispatcher-repo content, and default-work-dir creation fails loud if the workspace resolves under `~/.dreamux`. No state rebuild is required: existing TeamMate/Team records keep their persisted runtime paths and are read verbatim; only newly spawned/created agents that omit `repo` land in the new default location.
- Public MCP contract closeout (issue #199 Slice 1): teammate.spawn input is now name_prefix (returns the concrete name); team.create/status/history/dissolve/bind_group address by team_name, and team list/history rows are keyed by team_name (the duplicate team_id is dropped). teammate.history and team.history return { items, next_cursor } and keep the lifecycle status filter plus repo/since/until recovery dimensions, while dropping the legacy state/close_status filters, the id/team_id/display_name fields, the machine-local source_cwd/runtime_cwd/worktree fields, the Dreamux-made session_id, and runtime checkpoint from public history rows. The BREAKING/Rebuild upgrade note for this Epic lands in the final slice.
- Public MCP field collapse + repo input (issue #199 Slice 2): the shared teammate spawn/send/status/list output drops display_name, public role, team_id, and the runtime checkpoint; session_id now means the runtime-native thread id (early null acceptable), and the source_cwd/cwd/runtime_cwd/worktree family collapses into one compact `repo` view. team.status returns a team_name-keyed view without the duplicate team_id or machine-local repo_cwd/runtime_cwd/worktree. teammate.last no longer surfaces the internal ledger session_id. The public work-directory INPUT for teammate.spawn and team.create is now a single optional `repo` object ({ mode: reuse-cwd | managed, path?, base_ref?, branch?, slug?, cleanup? }; omitted uses the dispatcher's default directory), replacing the old cwd / repo_cwd / worktree inputs. The Dreamux-minted session ledger key is kept internally as a transitional detail and is retired with the Slice 3 records/turns storage. The BREAKING/Rebuild upgrade note for this Epic lands in the final slice.
- BREAKING: local storage model closeout (issue #199 Slice 3). Per-dispatcher TeamMate state moves to a per-name layout: `state/<dispatcher>/teammate/records/<name>.json` (the primary record: identity + a rolling recovery summary, the source for history/list/status) and `state/<dispatcher>/teammate/turns/<name>.jsonl` (the ONLY JSONL store: compact per-turn rows folded by `last`). The former `teammate/identities/<name>.json`, the per-dispatcher `teammate/sessions.jsonl` session ledger, and the `team/ledger/<team_name>.jsonl` team audit ledger are removed; team records and `team/channel-bindings.json` stay JSON. The Dreamux-minted `session_id` ledger key and the persisted `checkpoint` object are gone: `session_id` is now the runtime-native thread id, persisted directly, and the resume checkpoint kind is rebuilt from the runtime. 0.x is fail-loud + rebuild (no automatic migration). Rebuild: delete the old `~/.dreamux/state/<dispatcher>/teammate/identities/`, `~/.dreamux/state/<dispatcher>/teammate/sessions.jsonl`, and `~/.dreamux/state/<dispatcher>/team/ledger/` after upgrading, then close and respawn any teammates/teams to repopulate `teammate/records`, `teammate/turns`, and `team/records`.
- BREAKING: Team visibility closeout (issue #199 Slice 4). The `teammate.*` surface no longer leaks TeamLeaders or Team members to the dispatcher: visibility is enforced by one predicate at two scoped read chokepoints, so a dispatcher sees only the ordinary TeamMates it spawned, a TeamLeader sees only its own members, and an ordinary TeamMate sees no peers. A dispatcher inspects Teams through `team.*` compact summaries. The persisted channel binding key is renamed from `team_id` to `team_name` in `state/<dispatcher>/team/channel-bindings.json`. 0.x is fail-loud + rebuild (no automatic migration). Rebuild: delete `~/.dreamux/state/<dispatcher>/team/channel-bindings.json` after upgrading and re-bind any Team Feishu groups via `team.bind_group`.
- BREAKING: Pre-#199 local-state fail-loud closeout (issue #199 Slice 5). Dreamux 0.x does not migrate old TeamMate/Team state — a server that ran an earlier layout fails loud with explicit rebuild guidance. `dreamux serve` aborts startup (and `dreamux doctor` reports a matching diagnostic) when a dispatcher still has a removed whole-file/dir layout: `teammate/identities/`, `teammate/sessions.jsonl`, `teammate/history/`, or `team/ledger/`. A removed field still sitting in a present record is rejected by that record's reader: `checkpoint` / `checkpoint_kind` / `session_ref` / `display_name` / `close_status` on a `teammate/records/<name>.json`, or a `team/channel-bindings.json` row keyed by the old `team_id` instead of `team_name`. Detection only — the legacy paths/files are never read for migration, rewritten, or removed. Rebuild: after upgrading, delete the named path(s) under `~/.dreamux/state/<dispatcher>/` (run `dreamux doctor` to list them) and let the current `teammate/records/<name>.json` + `teammate/turns/<name>.jsonl` + `team/records/<team_name>.json` + `team/channel-bindings.json` layout rebuild; re-bind any Team Feishu groups via `team.bind_group`.

## 0.14.0
Thu, 11 Jun 2026 13:39:40 GMT

### Minor changes

- Logs stage + runtime socket path-budget fix (issue #182). Logs: runtime child stdout/stderr log files are opened eagerly as inherited fds and normal Codex/Claude traffic flows over the socket/stream, so they are usually empty; each supervisor now removes its child's stdout/stderr log on clean shutdown when it stayed zero-byte, so empty logs no longer accumulate one-per-start (files that captured startup/crash output are kept). Log retention stays MANUAL in 0.x — Dreamux does not age-prune logs; a 7-day retention is the documented guidance and zero-byte files are always safe to delete (see the dreamux-maintenance skill's Log Maintenance section). The whole ~/.dreamux/logs/ tree is rebuildable and safe to clear while no server runs. Runtime sockets: the volatile rendezvous-socket allocator now also considers a private per-user OS temp dir (resolved from TMPDIR/TMP/TEMP then os.tmpdir()) after $XDG_RUNTIME_DIR and ~/.dreamux/run/sockets/, but only when it is NOT a world-shared root like /tmp (Linux /tmp is still rejected). On macOS, os.tmpdir() is the per-user $TMPDIR (/var/folders/.../T, owner-only) and is far shorter than a long per-run $HOME, so Codex/Claude sockets stay within the Unix sun_path budget when there is no $XDG_RUNTIME_DIR and the run root would be over budget — fixing the macOS path-budget failure without using shared /tmp and without persisting socket paths. Rebuild: none. No config/state schema change; old empty log files left by a previous version may be deleted manually.
- BREAKING: dreamux-owned volatile run files moved from ~/.dreamux/state to ~/.dreamux/run (issue #182 PR-1): the admin socket (state/admin.sock -> run/admin.sock, plus its .lock) and the one-shot restart marker (state/restart-intent.json -> run/restart-intent.json). Codex app-server listen sockets no longer use descriptive in-state paths (state/<dispatcher>/codex.sock or teammate runtime dirs); every runtime start now allocates a fresh short random socket under $XDG_RUNTIME_DIR/dreamux/sockets/ or ~/.dreamux/run/sockets/, swept on server start, never persisted to durable state. Mixed-version caveat: a CLI or MCP shim older than this version looks for the admin socket under state/ and cannot reach a new server (and vice versa) — upgrade the package and restart the daemon so server and shims ship from the same version. Upgrade step: STOP the old daemon (dreamux daemon stop, or stop the managed service) BEFORE starting this version. Because the admin lock moved from state/admin.sock.lock to run/admin.sock.lock, a still-running old server would not be seen by the new one; the new server now fails loud on startup if a live old server still holds the legacy lock, so stop it first. The unused server.json path declaration was removed. Rebuild: nothing to rebuild; old leftovers may be deleted manually: ~/.dreamux/state/admin.sock, ~/.dreamux/state/admin.sock.lock, ~/.dreamux/state/restart-intent.json, ~/.dreamux/state/<dispatcher>/codex.sock, ~/.dreamux/state/<dispatcher>/teammate/runtime/<name>/codex.sock, and stale dreamux-codex-*.sock files under $XDG_RUNTIME_DIR (or the per-user os temp dir, e.g. macOS $TMPDIR).
- BREAKING: dreamux cache/spill artifacts moved into a new ~/.dreamux/cache tree (issue #182 PR-2). Teammate completion spill files moved from shared /tmp/teammate-<source>-<id>.output to ~/.dreamux/cache/<dispatcher-id>/spill/, and the Feishu inbound attachment cache moved from ~/.dreamux/state/<dispatcher-id>/feishu-attachments/ to ~/.dreamux/cache/<dispatcher-id>/feishu-attachments/. Both are rebuildable cache, not durable state: nothing reads a spill file back (only its path is inlined into a dispatcher turn) and attachments are re-fetchable. dreamux uninstall now also removes ~/.dreamux/cache. No automatic migration. Rebuild: nothing to rebuild; old leftovers may be deleted manually: stale /tmp/teammate-*.output files and the old ~/.dreamux/state/<dispatcher-id>/feishu-attachments/ directories.
- BREAKING (MCP contract): intent/note are now required on TeamMate and Team lifecycle tools (issue #182 PR-3). teammate.spawn requires intent and teammate.close requires note; team.create and team.create_group require intent and team.dissolve requires note. teammate.send gains an OPTIONAL intent that updates the recorded recovery subject before the turn. These are the durable recovery subject and the close/dissolve reason for the session/Team ledger. The synthetic 'team dissolved' fallback was removed — dissolve now records the operator's real reason. Persisted fields stay nullable, so existing identity/Team records written without intent/note still load (they read as null); only new lifecycle calls must supply the fields. A dispatcher/agent that called these tools without intent/note will now get a 'must be a non-empty string' rejection. The requirement is enforced at every layer — MCP json-schema, the shim arg builders, the admin methods, AND the TeamMate/Team service boundary — so even an in-process caller cannot persist a missing/empty required field. Recreating a closed Team now adopts the new create.intent (it was previously left stale). No durable file format or path change. Not included here: the later list/status/history reshape, ctx/history_events removal, or create/create_group merge (PR-7).
- BREAKING (dispatcher workspace + managed worktree layout): every configured dispatcher must now declare an explicit `cwd` (issue #182 PR-4). `dreamux serve` fails loud at startup if any enabled dispatcher has no `cwd` — there is no longer a fallback to a Dreamux state directory (`~/.dreamux/state/<id>/cwd`). A configured-but-missing `cwd` is created with mkdir -p; a `cwd` that is not a usable directory fails startup. `dreamux doctor` diagnoses the same contract per dispatcher (`dispatcher <id> workspace`). Rebuild: add `"cwd": "/abs/path/to/workspace"` to each dispatcher in ~/.dreamux/config.json (a real, operator-owned project directory, NOT inside ~/.dreamux) and `daemon restart`. Dreamux-managed TeamMate/Team Git worktrees were relocated OUT of `~/.dreamux/state/<id>/teammate/worktrees/` into the dispatcher workspace at `<cwd>/.workspace/worktree/<repo-disambiguated-slug>/<teammate-or-team-slug>/`. `.workspace/` is self-ignored (a `*` .gitignore) so managed worktrees never become repo content. The repo-disambiguated slug (`<sanitized-basename>-<sha256(repo-root):12>`) keeps same-named repos and Team/TeamMate worktrees distinct across repos. Managed worktree creation fails loud if the workspace resolves under `~/.dreamux`. Legacy teammate/Team identity records that still point at the old under-state worktree path are read verbatim (no rewrite, no deletion); only newly created managed worktrees use the new location, so a teammate with a since-deleted old managed worktree is re-prepared at the new path on reopen. reuse-cwd teammates are unchanged and do not require the dispatcher cwd contract. No persisted file FORMAT change. Not included: PR-5 session ledger, PR-6 list/status/history read surface, PR-7 Team MCP cleanup, logs/retention.
- Add durable TeamMate/Team session ledger capture (issue #182 PR-5). A new per-dispatcher append-only file `~/.dreamux/state/<dispatcher-id>/teammate/sessions.jsonl` records session lifecycle events (spawn/create, send, turn submitted — including TeamLeader turns delivered through a bound Team channel — turn settled, close/dissolve) keyed by a stable `session_id`. Each event denormalizes the facts needed to reconstruct work weeks later: repo, source cwd, runtime cwd, worktree slug/path/branch/base_ref, name, role (teammate/team_leader/team_member), team_id, human-readable leader name, intent, turn origin (dispatcher/team_leader/channel), runtime checkpoint kind + resumable session/thread id, status, and close note — never a volatile runtime socket path. TeamMate identity records gain a nullable `session_id` field; records written before this change read as null and the id is minted lazily on the first post-upgrade lifecycle event (spawn, send, channel turn, or close) and persisted, so existing teammates start being captured without a rebuild — and a send reopening a closed teammate reuses the existing id, so the key never re-keys to the runtime thread id. This is additive and backward compatible: no existing public MCP/tool behavior changes, the per-name history index and Team ledger are unchanged, and old state still loads. The capture is best-effort (a ledger write failure is logged, never failing a lifecycle verb); the settled-turn fact is captured after the reverse-delivery attempt regardless of its outcome, so a failed delivery still records recovery metadata and capture never perturbs delivery timing. No Rebuild required. Not included: the public `last(turns=N)`/filterable session read surface (PR-6), the `ctx`/`history_events`/Team-surface redesign (PR-6/PR-7), and any log relocation/cleanup.
- BREAKING: Concrete TeamMate names, durable last(turns), and read-surface cleanup (issue #182 PR-6, #188). The TeamMate `spawn.name` is now a requested base slug / display hint, NOT the final address: the service allocates a concrete, never-reused name (`${slug}-${suffix}` ordinary, `tm-${slug}-${suffix}` Team member, `tl-${team_slug}-${suffix}` TeamLeader; 8 base36-char suffix; slug truncated to keep the 64-char limit) and returns it as `teammate.name`. Callers MUST use the returned concrete name for every later send/status/last/close — the requested label is preserved as the new `display_name` field. Uniqueness is checked against all persisted identities (closed included), so a concrete name is never reused. A Team's durable `leader_name` is now the concrete `tl-` name instead of `${teamId}-leader` (which survives only as the leader's display_name); channel routing/status/dissolve read the stored name. The `last` verb is reworked: it reads a teammate's most recent settled turn(s) from the durable session ledger by concrete name, accepts `turns` (default 1, range 1..5), returns the final assistant output captured up to a 160,000-char hard cap with an `assistant_truncated` flag, and never starts/resumes a runtime — so it works for a closed/stopped teammate and is the fallback when reverse-delivery of a completion failed. The settled session-ledger event gains `assistant` + `assistant_truncated`; identity/status/ledger rows gain `display_name` (and `session_id` on ledger rows). The obsolete `ctx` and raw `history_events` verbs are removed from the TeamMate MCP schema, admin methods, and capabilities `verbs`; use `last` and `history` instead. Backward compatible state: identity records written before this change read `display_name` as null and remain usable without migration; the session ledger and per-name history index still load. Rebuild: none required for stored state — but any saved automation, prompt, or note that hard-codes a teammate name like `reviewer` or a Team leader name like `myteam-leader`, or that calls the removed `ctx`/`history_events` verbs, must be updated to use the concrete name returned by `spawn` and the `last`/`history` verbs.
- BREAKING: Align the Team MCP surface with the TeamMate read-surface model (issue #182 PR-7). The dispatcher-only `team` MCP is now addressed by Team `name` (the internal `team_id` storage key is unchanged and equal to `name` today), and its read tools mirror the TeamMate `list`/`status`/`history` split: `team.list` returns compact scan rows (name, status, intent, repo signal, leader name/state, member count, bound-group marker, timestamps) instead of full summaries; `team.status` returns one Team's detail by name including the active bound Feishu group; and the raw per-Team event `team.ledger` verb is replaced by a filterable `team.history` recovery search over Teams (filters: name, status, close_status, repo, intent text `grep`, `since`/`until`, `limit`, `cursor`) — the raw lifecycle event timeline stays internal. Binding is simplified: `team.bind_channel` becomes `team.bind_group` taking Team `name` + `chat_id` (Feishu group only; the redundant `chat_type` is removed from the public surface, which the binding store rejected for non-group anyway), and `team.transfer_channel_back` likewise drops `chat_type`. `team.create`/`team.create_group` are unchanged in this PR (create_group retirement is deferred to a follow-up). Rebuild: none for stored state — Team records, ledgers, and channel bindings load unchanged. But any saved automation or prompt that calls the Team MCP with `team_id`, or that uses the removed `team.ledger`/`team.bind_channel` verbs or a `chat_type` argument, must switch to addressing by `name` and to `team.history`/`team.bind_group` without `chat_type`.
- BREAKING: Final #182 cleanup — retire the public `team.create_group` tool and remove the write-only per-name TeamMate history index (issue #182 PR-8). (1) `team.create_group` (create a brand-new Feishu group and invite users) is removed from the Team MCP tool list, capabilities, admin methods, and docs. Its binding role is replaced by an optional `team.create` argument `bind_group: { chat_id }` that binds an EXISTING Feishu group chat to the new Team at create time; a standalone `team.bind_group` still binds an existing group later. The dreamux-side group-creation plumbing (TeamService.createGroup, the dispatcher's createFeishuGroup wiring, and the channel bot's createGroup method) was removed with it. (2) The per-name TeamMate history index `~/.dreamux/state/<dispatcher-id>/teammate/history/<name>.jsonl` is no longer written or read: since PR-6 the durable session ledger `sessions.jsonl` is the single recovery record (list/status/history/last all read it), so the per-name index had no readers and only added files. `appendHistory`, `TeamMateIdentityStore.history()`, the per-name path builder, and the `TeamMateHistoryEvent` types were removed. Rebuild: none required for stored state — Team records, channel bindings, and the session ledger load unchanged. Existing `teammate/history/<name>.jsonl` files become unused and may be deleted manually. Any saved automation or prompt that called `team.create_group` must switch to `team.create` with `bind_group` (binding an existing group); creating a brand-new Feishu group from the Team MCP is no longer supported.

## 0.13.0
Wed, 10 Jun 2026 07:24:34 GMT

### Minor changes

- Add runtime-scoped Claude Code Remote Control and make TeamMate capabilities report spawnable agent runtime ids.
- completion delivery now returns accepted at engine-take instead of after classifying the turn's model outcome (fixes over-delivery + removes turn-duration-coupled accept latency); subtype now authoritative in claude result classification
- TeamMate MCP parity PR1 (issue #126): API/ledger foundation and event/wait contract, no worker execution yet and no tm wrapping. Bump the TeamMate task ledger to v2 with separated lifecycle_status/delivery_status, a monotonic per-task event stream, a steerable inputs[] session, and close/runtime/target/provider_ref/intent/operation_id plus reserved Team Mode fields; v1 task files still load via lossless in-memory migration and only an unknown future version fails loud (no rebuild required). Add dispatcher-facing MCP tools run_task, execute_task, send_input, await_completion, and get_capabilities (existing schedule/list_tasks/get_task/pull_result stay compatible); without a worker the execution tools report provider_unavailable and send_input queues input. Add a server-owned bounded event/wait broker so await_completion returns a structured still_running on timeout instead of failing.
- TeamMate MCP parity PR2 (issue #126): add the worker provider seam and an in-memory fake provider, with no real Codex/Claude Code worker yet and no tm wrapping. Introduce a runtime-agnostic TeamMateWorkerProvider (per-task steerable session, distinct from the dispatcher's long-lived AgentRuntimeProvider) plus a permissive worker catalog; a provider drives lifecycle through callbacks and never writes the ledger. Add a TeamMateWorkerExecutionService that maps onRunning/onCompleted/onFailed/onCancelled onto ledger transitions (markRunning, the PR1 record-before-deliver completion path, and a cancelled close) with wait-broker notifications, idempotent execution, and no re-execution of terminal tasks. Wire run_task/execute_task through the service, route send_input to a live worker session (promoting a queued input to submitted via the new ledger.markInputSubmitted), and make the worker catalog the source of truth for get_capabilities. Production behaviour is unchanged from PR1: with the default empty worker catalog every provider is worker-unavailable and execution tools still report provider_unavailable; only an injected fake catalog (tests) runs a session.
- TeamMate MCP parity PR8 (issue #126): make the push-delivery model real. (1) Codex worker turn completion is robust to a single-thread threadId field-shape drift (acceptAnyThread), and a redacted Codex WS event trace is captured to a new get_task_logs 'events' stream and summarized in the task failure text — a stalled turn now names the likely cause (e.g. no model output => worker auth/network/quota/proxy) instead of an opaque timeout. PR7's bounded turn timeout is preserved. (2) await_completion is removed as a dispatcher-facing MCP tool; normal orchestration is run_task -> the dispatcher turn ends -> the server delivers/wakes a new turn, with get_task/pull_result as the recovery/read path. The base prompt and bundled dispatcher skills are updated accordingly; an internal wait primitive remains for tests/admin diagnostics only. (3) The managed-service unit PATH now also includes the Claude Code install directory so server-hosted builtin:claude-code workers can resolve `claude` (DREAMUX_CLAUDE_BIN overrides; best-effort, codex-only installs are unaffected). BREAKING: the dispatcher-facing await_completion MCP tool is gone — a caller that invokes it now gets an unknown-tool error; re-read results with get_task/pull_result instead. Rebuild: run `dreamux daemon install` (or `dreamux onboard`) to regenerate the service unit PATH so `claude` resolves for builtin:claude-code workers, then `dreamux daemon restart` to pick up the updated base prompt and bundled skills.
- BREAKING: Feishu is now a built-in bidirectional channel rather than a provider-registry channel implementation. Channel plugin loading remains interface-only; the provider registry now only backs Agent Runtime providers. DispatcherService owns dispatcher agent lifecycle, live runtime slots, restart-notice injection, and Feishu channel MCP dispatch; server.ts is wiring only.
Rebuild: restart dreamux serve after upgrading so dispatcher agents and the built-in Feishu channel are recreated under DispatcherService ownership.
- Load installed npm Agent Runtime providers from dispatchers[].runtime.provider before config validation. External runtime providers now enter through the same provider registry, AgentRuntimeProvider catalog, DispatcherService creation path, and provider-owned capability declaration contract as builtin runtimes. Missing packages, missing exports, invalid provider contracts, incomplete capability declarations, and non-runnable descriptors fail loudly with the selected provider ref.
- A builtin:claude-code dispatcher now receives a dreamux-adapted dispatcher role prompt, injected via the resident `claude --append-system-prompt` flag (layered on top of claude's own system prompt). Previously claude dispatchers got no dispatcher prompt at all; they now carry the coordinator-not-worker boundaries, TeamMate MCP + push-delivery model, Feishu visible-reply contract, and public-artifact safety rules. No config or state change; the prompt is applied at dispatcher (re)start, so restart claude dispatchers to pick it up.
- Wire teammate -> dispatcher reverse completion delivery (issue #147): a finished teammate's result is now delivered back to the dispatcher as a new turn via the runtime's completionInput, using each engine's native mechanism (codex thread/inject_items + a trigger turn; claude-code native <task-notification> with isSynthetic). A teammate turn that errors at the model level (codex fatal 'error' notification) is delivered as 'failed' instead of hanging. NOTE: codex reverse-delivery requires codex 0.137+ (thread/inject_items); on older codex it fails loudly with a 0.137 hint rather than dropping the completion silently. Also fixes spawning a cross-provider teammate (e.g. a builtin:claude-code teammate under a builtin:codex dispatcher), which previously threw 'is not wired to Claude Code'.
- BREAKING: doctor now resolves each dispatcher's runtime through its agent-runtime provider's self-reported diagnostic (issue #146 fold) instead of branching on the codex/claude provider ref. The codex diagnostic adds a hard version floor: doctor fails loud when the resolved Codex binary is below 0.137.0, because teammate-completion delivery appends to the dispatcher thread via thread/inject_items, an RPC that exists only on codex >= 0.137 (issue #147). The claude per-dispatcher detail line changed from 'does not use Codex home state' to a neutral 'no host-managed home state'.
Rebuild: if `dreamux doctor` reports a Codex version below 0.137.0, upgrade Codex (the codex-cli binary referenced by the agent's config.bin / CODEX_HOST_CODEX_BIN) to >= 0.137.0; no config file change is required.
- BREAKING: ~/.dreamux/config.json runtime config is hoisted into named top-level agents[]. Each dispatcher now references a runtime by id via dispatchers[].agentRuntime instead of carrying an inline dispatchers[].runtime block; config lives only in agents[] ({ id, provider, config }). The old inline-runtime shape, a dispatcher missing agentRuntime, a dangling agentRuntime id, and a duplicate agents[].id all fail loud at load with rebuild guidance (no migration shim). Each agent's config is parsed through its provider's readConfig.
Rebuild: rewrite ~/.dreamux/config.json to the new schema (declare runtimes under agents[] and reference them with dispatchers[].agentRuntime), then restart dreamux serve. To regenerate from scratch instead, first MOVE ASIDE or DELETE the old config.json -- onboard loads the existing config and would otherwise fail loud on the old inline-runtime shape before it could rewrite -- then run dreamux onboard.
- BREAKING: a teammate now references a named agent (an agents[].id) instead of a provider. spawn takes agent_runtime (was provider_ref); the teammate resolves that id against the global agents[] map into its own { provider, config }, so a teammate on a different agent than its dispatcher (e.g. a claude teammate under a codex dispatcher) runs with its OWN config and the cross-provider 'is not wired to ...' mismatch is gone. Persisted teammate identity records change their field provider_ref -> agent_runtime; a pre-#148 identity fails loud on the next lifecycle verb with rebuild guidance (no migration shim). A teammate whose agent_runtime no longer matches any agents[].id also fails loud on resume rather than silently defaulting a runtime.
Rebuild: close and respawn affected teammates, or delete the stale identity files under ~/.dreamux/state/<dispatcher>/teammate/identities/<name>.json, then respawn. Ensure each teammate's agent_runtime names an existing agents[].id in ~/.dreamux/config.json.
- BREAKING: Remove the teammate `resume` verb for issue #155. The `resume` MCP tool (dispatcher-scoped `teammate` server) and the `mcp.teammate.resume` admin method are gone; callers must use `send`. `send` now subsumes resume semantics: when the named teammate is not live — including one previously `close`d — send reopens it from its persisted checkpoint (clearing the closed markers) and then submits, instead of failing with "TeamMate ... is closed". `close` is now a reversible soft-stop (it still records closedAt/note, but a later `send` revives the teammate). Read-only verbs (last/ctx/status) still fail-loud on a closed teammate and never silently reopen it. No on-disk migration is required: existing closed teammate identities and pre-#155 history files (which may contain `"type":"resume"` events) remain readable; the reopen path is purely runtime behavior.
- BREAKING: TeamMate MCP/admin spawn now requires `cwd`; the service no longer falls back to a dispatcher cwd or Dreamux runtime directory for native teammates. Call `spawn({ name, prompt, cwd, worktree?, agent_runtime? })` and pass either `worktree: { mode: "reuse-cwd" }` or `worktree: { mode: "managed", slug?, base_ref?, branch?, cleanup? }` for Dreamux-managed worktree isolation. TeamMate identities now persist dispatcher owner metadata plus source/runtime cwd and worktree cleanup metadata. Rebuild: respawn any caller-side TeamMate automation that omitted `cwd`; old identity files without owner/worktree metadata still read as dispatcher-owned reuse-cwd records and are rewritten only on later lifecycle mutation.
- BREAKING: TeamMate MCP/admin `history` now returns bounded session ledger rows (`items`, `next_cursor`) across TeamMate identities instead of the raw event list for one required name. Use `history_events({ name })` for the raw forward-only per-TeamMate timeline. Ledger rows expose agent runtime id, dispatcher owner metadata, source/runtime cwd, managed worktree metadata, close/cleanup state, intent, prompt previews, and a structured `send` resume hint. Rebuild: update dispatcher-side automation that parsed `history({ name }).events` to call `history_events({ name })`, or switch it to the ledger row shape.
- BREAKING: Team Mode adds persistent Feishu group channel binding. Dispatcher-scoped `team` MCP/admin now includes `bind_channel` and `transfer_channel_back`; bound group inbound is routed to the TeamLeader after the normal Feishu gate/format path, while unbound and P2P inbound continue to the dispatcher. TeamLeader Feishu reply/react calls are scoped to bound team channels. Rebuild: update Team Mode dispatcher automation to call `team.bind_channel({ team_id, chat_id, chat_type: "group" })` for group handoff and `team.transfer_channel_back({ chat_id, chat_type: "group" })` before returning a group to the dispatcher.
- BREAKING: Team Mode adds dispatcher-only `team.create_group` for P2P control requests. The command creates a Team, asks the shared dispatcher Feishu bot to create a group and invite requested peers, then binds that new group to the TeamLeader; the source P2P channel remains routed to the dispatcher. Rebuild: update Team Mode dispatcher automation to call `team.create_group({ name, repo_cwd, leader_agent_runtime, source_chat_id, source_chat_type: "p2p", requester_open_id, invite_open_ids? })` when a P2P requester asks for a team group. TeamLeaders still use the shared bot and do not receive independent Feishu credentials.
- BREAKING: Team Mode core adds dispatcher-scoped `team` MCP/admin lifecycle methods (`create`, `list`, `status`, `ledger`, `dissolve`) and extends TeamMate identities with role/owner/team metadata. TeamMate MCP calls are now scoped by a server-derived caller principal: dispatchers see dispatcher-owned ordinary TeamMates and TeamLeaders, while TeamLeaders see only their own team-owned members. Rebuild: update dispatcher automation to use `team.create({ name, repo_cwd, leader_agent_runtime, ... })` for TeamLeader creation and avoid passing owner/team fields through `teammate.*`; those fields are derived by Dreamux.
- TeamMate codex runtimes now inherit the parent dispatcher's codex config (approval_policy / sandbox_mode / extra_args) instead of always-defaults — consistent with the already-inherited bin/env/timeout. A codex teammate under a codex dispatcher now runs with that dispatcher's sandbox/approval/args.
- TeamMate codex workers no longer receive the dispatcher base system prompt. Previously the shared codex runtime hard-coded the dispatcher instructions on every thread start, so teammate workers were wrongly told they were the dispatcher; the dispatcher prompt is now supplied only to the dispatcher agent via the runtime's systemPrompt capability. A teammate's role/task continues to arrive through its first turn. Restart dispatchers to apply.
- Provider architecture PR A (issue #135): replace the old capability registry with a provider-ref registry that validates builtin refs and provider kind only. Runtime and channel provider factories now receive descriptors resolved from the server-owned registry, and config provider validation uses the injected registry before Phase 1 wired-provider checks. BREAKING: the exported CapabilityRegistry/capability descriptor API is removed; use ProviderRegistry for ref/kind lookup and read capabilities from provider instances.
- Realign the AgentRuntime contract around submitTurn and provider-owned capabilities, and make the agent runtime catalog read implementations from the provider registry instead of carrying its own provider map.
- Move TeamMate orchestration into Dispatcher Service and slim server wiring.
- BREAKING: Replace the TeamMate task/worker MCP model with named, resumable TeamMate agents. The dispatcher-scoped teammate MCP now exposes spawn/send/resume/close/history/list/status/last/ctx/get_capabilities; schedule/run_task/execute_task/send_input/cancel_task/get_task_logs/list_tasks/get_task/pull_result and task_id records are removed.

Rebuild: remove old ~/.dreamux/state/<dispatcher-id>/teammate/ledger.json and ~/.dreamux/state/<dispatcher-id>/teammate/tasks/ state, then recreate TeamMate sessions with the new spawn/resume flow. TeamMates now reuse AgentRuntime providers with async identity/history persistence.

### Patches

- Fix the built CLI cold-start crash (Cannot access 'BUILT_IN_DEFAULTS' before initialization) introduced by the #148 agents[] refactor. config/config.ts no longer registers the builtin runtime catalog itself; builtin runtimes are now composed by the caller through loadConfigWithBuiltins, which removes the static import cycle (config -> catalog -> builtin -> platform/paths -> config) at its root rather than deferring the temporal-dead-zone read. Adds a built dreamux --version smoke gate before CI and release publish steps.
- Fix Codex socket startup failure under deep state roots (long $HOME, long dispatcher/teammate names). When the descriptive socket path (state/<dispatcher>/.../codex.sock) exceeds the Unix sun_path budget, the socket now falls back to a short deterministic path under a private per-user runtime dir (XDG_RUNTIME_DIR, or a non-shared os tmpdir such as the macOS per-user $TMPDIR) instead of failing to start. The shared /tmp is never used; with no private root available the start still fails loudly. No action needed for existing installs whose paths fit the budget.
- TeamLeader completion routing is now per turn instead of per role: bound-channel leader turns stay pull-only (team ledger), while dispatcher-initiated leader send/control turns return to the dispatcher as completions; removes the redundant getLast-polling completion path that double-delivered leader ledger rows and team-member completions
- Fix TeamMate reverse-completion delivery so duplicate/retried completions are idempotent and multi-send steering shares one current-turn completion.
- Add the server-hosted TeamMate scheduling MCP and versioned per-dispatcher task ledger for issue #110.
- Add TeamMate completion delivery (Codex inbox/turn + Claude Code task notification), bounded retry to an inspectable delivery_failed/pull-available state, and read-only result retrieval MCP tools for issue #110.
- Stabilize issue #110 closure with providerized README/help updates, doctor coverage for Claude Code runtime diagnostics, and a durable Epic closure checklist.
- Run builtin:claude-code as a resident stream-json process (one long-lived `claude --print --input-format stream-json` child per dispatcher) instead of a one-shot `claude --print` per turn, for issue #120. Adds an optional `dispatchers[].runtime.config.turn_timeout_ms` (default 600000) bounding a single resident turn, and the logs/claude-code/ stderr log directory; persisted state schemas are unchanged and the new config field is optional with a default.
- Align bundled dispatcher skills and the injected dispatcher base prompt with the server-hosted TeamMate MCP (schedule/list_tasks/get_task/pull_result) as the primary scheduled-task interface, keeping the tm CLI as a labeled fallback and stating the #110 Phase 1 boundary.
- Add the real builtin:claude-code TeamMate worker provider (issue #126 PR4): TeamMate MCP now executes Claude Code tasks too, selectable by pinning provider_ref=builtin:claude-code (default routing stays builtin:codex). get_capabilities reports both workers (claude-code is single-turn, steer:false). Adds server-owned per-task worker paths state/<id>/teammate/workers/<taskId>.mcp.json and logs/claude-code/teammate/<id>/<taskId>.stderr.log (auto-created, no operator action). No config schema or persisted-format change.
- Add TeamMate MCP control/query parity (issue #126 PR5): two additive tools so a dispatcher can stop and inspect a worker over MCP without shelling out to a tm CLI, killing a process, or tailing a log file. cancel_task stops a live worker (reaping its resources) or closes a not-yet-running/orphaned task in the ledger as cancelled, and is an idempotent no-op on an already-terminal task. get_task_logs returns a bounded tail of a worker's diagnostic logs (Codex app-server stdout protocol frames + stderr; Claude resident-child stderr) for the EFFECTIVE worker, resolving the catalog default for tasks that did not pin a provider. Read/wait parity (status/history/last/poll) was already served by list_tasks/get_task/pull_result/await_completion; resume and multi-turn stay deferred (one-turn workers). No config schema, persisted format, or runtime path change.
- Align bundled dispatcher skills and the injected dispatcher base prompt with the real TeamMate MCP surface (issue #126 PR6). After PR3-5 wired real workers, the docs still carried the stale #124 'Phase 1 boundary' caveat claiming the MCP could not run a scheduled task to completion and that the tm CLI was the only executed-result path. They now present the server-hosted TeamMate MCP as the default orchestration interface: run_task/execute_task execute the default builtin:codex worker for real (builtin:claude-code via provider_ref, single-turn), list_tasks/get_task/pull_result/await_completion read and wait without polling, and cancel_task/get_task_logs/get_capabilities control and inspect a worker. The tm CLI is the explicit fallback for resume, multi-turn continuation, dead-session recovery, and isolated managed worktrees, which the in-place MCP workers do not yet cover. Documentation/template realignment only: no config schema, persisted format, runtime path, or MCP tool behavior change, and the symlinked skills pick up the new text on the next serve with no rebuild required.
- Fix two TeamMate installed-state validation blockers (issue #126 PR7). (1) get_capabilities no longer claims a worker is available when its binary cannot be started in the dispatcher service environment: it now probes each built-in worker's binary on the service PATH and reports worker_available:false with a reason for a wired-but-unresolvable worker (the builtin:claude-code ENOENT case where `claude` is absent), with execution_available derived from the probed rows. The probe runs only on the advertisement path; the execute path already returned structured provider_unavailable on spawn. (2) A builtin:codex worker turn that reaches running but never completes (a stall in turn execution — auth, network, or model quota) is now bounded by a new turn_timeout_ms: on expiry the task fails with a self-contained message instead of sitting running forever with an empty diagnostic log. Config: turn_timeout_ms is added to the codex runtime config (dispatchers[].runtime.config.turn_timeout_ms), additive and defaulted to 600000ms, so existing configs load unchanged; `dreamux onboard` now writes it. No persisted-format or runtime-path change, and the tm fallback path is unchanged.
- Refactor builtin Agent Runtime module layout without behavior changes: move the Codex AgentRuntime implementation under agent-runtime, split the Claude Code resident transport into claude-code supervisor/rpc/stream/types/mcp-config modules, and share process-group helpers from runtime/process.
- Change the builtin:claude-code per-turn deadline from a fixed total-turn wall-clock to an idle/inactivity window for issue #156. `turn_timeout_ms` (default 600000) is now reset on every inbound stream line, so it bounds the max time the resident child may emit no stream activity rather than the total turn duration. A long but continuously-streaming turn (e.g. a deep audit running many tool calls for far longer than the window) is no longer reaped mid-work, while a genuinely wedged child that goes silent for the whole window is still failed and reaped (preserving the #120 anti-hang intent). No config change is required; the field name and default are unchanged, but its semantics changed (total-turn cap -> idle window). The timeout error text changed from 'timed out ... without a result' to 'stalled: no stream activity for {ms}ms'.
- Bound teammate completion push-back and move channel-input assembly into each runtime (#164). Completion results over an inline budget (default 32000 chars, TASK_MAX_OUTPUT_LENGTH override, clamped to 160000) are spilled to a 0600 /tmp file and only the path is inlined. The claude-code runtime drops the fake <task-notification> XML and delivers a plain status-varied user turn (isSynthetic:false; capability kind claudeCodeTaskNotification -> claudeCodePlainTurn); codex keeps its <teammate_session_completion> wrapper. CompletionEnvelope.status widens to completed|failed|stopped (no longer folded). Inbound Feishu messages are now assembled by each runtime into the native <channel source="feishu" …> envelope (codex's <feishu_message> wrapper retired); the channel layer stops pre-rendering XML and hands runtimes neutral structured pieces. No persisted config/state format changes, so no rebuild is required.
- Remove the in-memory DispatcherRow field codex_cwd and the server-ctl `codex-cwd` status flag (the field was never persisted to status.json). The dispatcher runtime cwd is now a required launch parameter supplied by the Dispatcher Service; codex bin resolution and per-runtime artifact paths moved behind the Agent Runtime providers. No persisted format change; no rebuild required.

## 0.12.0
Sat, 06 Jun 2026 10:02:33 GMT

### Minor changes

- BREAKING: remove the top-level `codex` block from ~/.dreamux/config.json. All Codex settings are now dispatcher-local under dispatchers[].codex (bin, approval_policy, sandbox_mode, extra_args, extra_env, initialize_timeout_ms), each with a built-in default (codex / never / workspace-write / [] / {} / 10000) so the whole codex object can be omitted. The server uses each dispatcher's own codex.bin and initialize_timeout_ms. CODEX_HOST_CODEX_BIN remains an optional host-level override of the codex binary for every dispatcher; onboard no longer auto-bakes it into the managed-service unit (the unit PATH carries the codex dir instead, so dispatcher-local codex.bin is authoritative). A config that still has a top-level `codex` block fails loud on load. Rebuild: edit ~/.dreamux/config.json — delete the top-level `codex` block and move any approval_policy/sandbox_mode/extra_args/bin into the relevant dispatchers[].codex; then `dreamux daemon restart` (re-run `dreamux onboard` if you want the new dispatcher-local bin re-derived into the service PATH). Existing service units that still set CODEX_HOST_CODEX_BIN keep working — there it stays the override and nothing breaks.

## 0.11.3
Sat, 06 Jun 2026 09:58:26 GMT

### Patches

- Fix team-dev-workflow skill frontmatter so Codex can parse it. The description had an unquoted colon-space ("dispatcher: adversarial ..."), which strict YAML reads as a nested mapping and rejects with "mapping values are not allowed here", making the bundled skill silently invisible in Codex. Reworded the description to drop the colon; the skill now lists and parses.

## 0.11.2
Sat, 06 Jun 2026 08:02:51 GMT

### Patches

- Add a Dreamux dispatcher base prompt for Codex app-server thread start/resume and update @excitedjs/tm to 2.4.1

## 0.11.1
Fri, 05 Jun 2026 16:51:01 GMT

### Patches

- Remove dead 0.x compatibility shims (issue #98). Delete the old copied-dispatcher-skill -> bundled-symlink fingerprint migration: a real directory at a bundled skill path is now always left untouched ('skipped'). Rebuild: if a dispatcher workspace still has an old hand-copied skill directory under .codex/skills/, remove or rename it so startup recreates the bundled symlink. Also delete the dead runtime_dir leftovers: the runtimeRoot() alias, the onboard runtimeDir answer, and the CLI --runtime-dir option, which previously was accepted-and-ignored and now fails loud as an unknown argument. No change to bundled-skill install/update, service unit re-registration, or service Node path selection.

## 0.11.0
Fri, 05 Jun 2026 16:16:45 GMT

### Minor changes

- Unify persisted-file version policy (issue #98). BREAKING: dispatcher access.json is now v2-only; the legacy v1 shape (dm.allow_users + group.follow_users) is no longer auto-migrated and an unsupported/missing version fails loud. Rebuild: delete the dispatcher's access.json to return to the secure default (no one authorized), then recreate it as a v2 access.json with allow_users and group.policy (see the access.json section in the dreamux README) and restart; note that `dreamux onboard` does not restore access grants. status.json and restart-intent.json now warn-and-rebuild / warn-and-drop on incompatible, malformed, or invalid-field content instead of silently discarding or misreading; neither hard-fatals the server.

## 0.10.0
Fri, 05 Jun 2026 15:34:49 GMT

### Minor changes

- Add the `dreamux changelog` command (and `--json`) that prints the installed package's bundled CHANGELOG, and ship CHANGELOG.md/CHANGELOG.json in the package files. This is the upgrade-time information entry point for the 0.x fail-loud + rebuild policy (issue #98).

## 0.9.8
Fri, 05 Jun 2026 14:06:54 GMT

### Patches

- Trusted peer-bot inbound now requires both a trusted sender open_id and an @-mention of this bot; introduce trusts only mention open_id (no union_id/user_id fallback); add diagnostic-only sender_union_id to inbound-drop logs (issue #102)

## 0.9.7
Fri, 05 Jun 2026 12:14:20 GMT

### Patches

- Restructure the bundled dispatcher skill into a router plus references, make the teammate engine a deliberate explicit choice instead of forcing codex, and add prompt-composition, router-posture, and inspect/resume workflow guidance.

## 0.9.6
Fri, 05 Jun 2026 08:05:04 GMT

### Patches

- Fix the published npm install path by removing the ahead-of-use Feishu channel runtime dependency.

## 0.9.5
Fri, 05 Jun 2026 05:41:43 GMT

### Patches

- Bundle the dispatcher, team-dev-workflow, and dreamux-maintenance skills and install them as workspace-local symlinks for each dispatcher.

## 0.9.4
Fri, 05 Jun 2026 05:30:23 GMT

### Patches

- Route Feishu inbound formatting through feishu-channel, including downloaded attachment paths and fallback resource references.

## 0.9.3
Fri, 05 Jun 2026 03:34:03 GMT

### Patches

- Fix group /introduce authorization to follow the group policy: under follow-user it now ignores allow_chats and gates only on allow_users, matching the delivery gate; block is denied explicitly (group_blocked); allowlist is unchanged.

## 0.9.2
Fri, 05 Jun 2026 02:47:01 GMT

### Patches

- Add a best-effort Feishu channel acknowledgement for authorized group /introduce commands.

## 0.9.1
Thu, 04 Jun 2026 23:08:48 GMT

### Patches

- Remove all synchronous blocking IO from package source (fs/promises + async child_process) and add a permanent ESLint gate (n/no-sync + import/syntax backstops via the shared @excitedjs/eslint-config) wired through rush lint, CI, and the pre-commit hook (issue #85)

## 0.9.0
Thu, 04 Jun 2026 20:50:14 GMT

### Minor changes

- Add daemon command group (install/uninstall/start/stop/restart), enable systemd linger so the user service starts at boot, and inject a restart-completed notice into resumed dispatchers after daemon restart --notify-resumed

## 0.8.0
Thu, 04 Jun 2026 20:24:45 GMT

### Minor changes

- Fix the follow-user group-access semantics: the dispatcher runtime gate (dreamuxFeishuGate) now gates group delivery on a single global allow-user list shared with direct messages, instead of a separate group.follow_users list, so a sender on the global allowlist who @-mentions the bot is delivered in any group (issue #79). The access.json shape is unified to v2: a top-level allow_users list plus an explicit group.policy (block | allowlist | follow-user); v1 files are migrated forward by readDispatcherAccess (legacy dm.allow_users and group.follow_users are merged and de-duplicated, the policy is inferred, and the first save rewrites the file). An empty allow_users now authorizes nobody, consistent with direct messages. /introduce sender authorization moves to the global allow_users list while still requiring the chat to be named in allow_chats.

## 0.7.0
Thu, 04 Jun 2026 20:18:37 GMT

### Minor changes

- Prefer a stable platform-aware system Node for the managed service (Homebrew on macOS, system paths on Linux) with fallback to the current Node, and add a non-fatal dreamux doctor advisory when the service Node is bound to a version manager.

## 0.6.2
Thu, 04 Jun 2026 19:41:43 GMT

### Patches

- Log a distinct channel diagnostic ('introduce detected but not authorized') with a stable reason code (non_group / empty_sender_id / chat_not_allowlisted / sender_not_followed) when a group /introduce is detected but the sender is not authorized, instead of letting it surface as an ordinary gate drop (e.g. 'bot not mentioned'). Gate, trust, and /introduce semantics are unchanged (issue #77).

## 0.6.1
Thu, 04 Jun 2026 18:47:15 GMT

### Patches

- Inject each dispatcher's per-dispatcher channel logger into its Feishu bot/transport, so the transport's Lark SDK and WebSocket connection diagnostics land in logs/feishu-channel/<id>.log alongside the host's own channel decisions (issue #74).

## 0.6.0
Thu, 04 Jun 2026 17:58:38 GMT

### Minor changes

- Add persistent structured file logging (pino) across server, Feishu channel, gate/drop/inbound/outbound/introduce, dispatcher runtime, and the feishu-mcp stdio shim; logs persist under ~/.dreamux/logs (issue #70).

## 0.5.0
Thu, 04 Jun 2026 17:12:55 GMT

### Minor changes

- Inject a one-shot <group_bots> context of a group's trusted bots on the first delivered message after /introduce (commit-after-notify, generation-safe clear); add a model-facing list_chat_bots MCP tool (backed by a read-only mcp.list_chat_bots admin method) returning a chat's known + trusted bots; and change the inbound reaction lifecycle to add-then-cancel so the message never shows a zero-reaction window during the received -> in-progress transition.

## 0.4.0
Thu, 04 Jun 2026 15:47:28 GMT

### Minor changes

- Add the Feishu event-registry seam (FeishuBot.start now takes a route object with onMessage + optional onBotMemberAdded) and the group /introduce hard contract: /introduce triggers only when the sender is allowlisted, with no @-mention of the bot required. A new chat-bots.json store separates passive bot awareness from introduced trust.

## 0.3.3
Thu, 04 Jun 2026 14:09:37 GMT

### Patches

- Fix Feishu inbound reaction emoji type values.

## 0.3.2
Thu, 04 Jun 2026 13:15:17 GMT

### Patches

- Submit accepted Feishu inbound with non-blocking turn/start delivery and three-state reactions.

## 0.3.1
Thu, 04 Jun 2026 07:44:14 GMT

### Patches

- Fix managed service startup when Node is provided by nvm.

## 0.3.0
Thu, 04 Jun 2026 05:00:52 GMT

### Minor changes

- 调整 onboard 与 dispatcher runtime，使其继承本机 Codex 状态，改用 JSON 配置并新增 uninstall 指令。

## 0.2.0
Wed, 03 Jun 2026 07:57:13 GMT

### Minor changes

- 调整 onboard 与 dispatcher runtime，使其继承本机 Codex 状态，改用 JSON 配置并新增 uninstall 指令。

## 0.1.4
Wed, 03 Jun 2026 04:29:43 GMT

### Patches

- Fix onboard Codex marketplace installation from the public dreamux repository.

## 0.1.3
Tue, 02 Jun 2026 18:55:21 GMT

### Patches

- Implement dreamux onboard: first-run wizard, dispatcher-private Codex home setup, plugin installation, service registration, and transparent file ledger output.
- Add the issue #18 dreamux serve foundation: single global bin command tree, dispatcher-private Codex homes, and serve-time Codex home checks.

## 0.1.2
Sun, 31 May 2026 07:02:52 GMT

### Patches

- Thread Feishu replies and drop bot-loop inbound messages

## 0.1.1
Sat, 30 May 2026 17:49:32 GMT

### Patches

- init

