# Change Log - @excitedjs/dreamux

This log was last generated on Wed, 10 Jun 2026 07:24:34 GMT and should not be manually modified.

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

