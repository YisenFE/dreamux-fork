# Change Log - @excitedjs/feishu-transport

This log was last generated on Thu, 04 Jun 2026 23:08:48 GMT and should not be manually modified.

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

