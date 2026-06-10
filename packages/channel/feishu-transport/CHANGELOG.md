# Change Log - @excitedjs/feishu-transport

This log was last generated on Wed, 10 Jun 2026 07:24:34 GMT and should not be manually modified.

## 0.3.0
Wed, 10 Jun 2026 07:24:34 GMT

### Minor changes

- Add Feishu group creation and member-invite transport APIs used by Dreamux Team Mode create_group. The APIs fail loudly when the installed Feishu SDK/client does not expose the required chat methods.

## 0.2.3
Fri, 05 Jun 2026 14:06:54 GMT

### Patches

- narrowMetaFromEvent surfaces a diagnostic sender_union_id from the inbound event; it is observability-only and never used for access matching (issue #102)

## 0.2.2
Fri, 05 Jun 2026 05:30:23 GMT

### Patches

- Expose structured inbound resources and a raw message-resource fetch seam for channel-owned attachment handling.

## 0.2.1
Thu, 04 Jun 2026 23:08:48 GMT

### Patches

- Adopt the shared @excitedjs/eslint-config flat config and the synchronous-blocking-IO lint gate (issue #85); no runtime change

## 0.2.0
Thu, 04 Jun 2026 18:47:15 GMT

### Minor changes

- Add an explicit, additive `logger?` option to `FeishuTransportOptions` (a package-owned minimal `TransportLogger` interface) so a host can fold the transport's own diagnostics — Lark SDK logging, WebSocket connection lifecycle, and best-effort doc-comment/metadata/bot-info/socket-close failures — into its per-component log. Instance-level: each transport derives its SDK and connection sinks from the injected logger. With no logger the historical stderr behavior is preserved byte-for-byte (issue #74).

## 0.1.0
Thu, 04 Jun 2026 05:00:52 GMT

### Minor changes

- Export the access-state persistence contract used by host channel gates.

## 0.0.2
Sun, 31 May 2026 07:02:52 GMT

### Patches

- Add core parsing helpers for Feishu bot-member-added events and mention names.
- Thread Feishu replies with outbound targets

## 0.0.1
Sat, 30 May 2026 17:49:32 GMT

### Patches

- init

