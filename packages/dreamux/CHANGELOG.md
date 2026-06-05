# Change Log - @excitedjs/dreamux

This log was last generated on Fri, 05 Jun 2026 03:34:03 GMT and should not be manually modified.

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

