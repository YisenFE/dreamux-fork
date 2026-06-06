# Change Log - @excitedjs/dreamux

This log was last generated on Sat, 06 Jun 2026 10:02:33 GMT and should not be manually modified.

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

