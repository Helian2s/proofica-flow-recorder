# Privacy and redaction

[← Repository overview](../README.md)

Flow Recorder includes input-value filtering, but the current prototype does not provide complete redaction across the recorded session. Use synthetic data on local test pages. This document distinguishes the implemented value pipeline from the gaps in other capture paths.

## Input-value handling

The [redaction engine](../packages/recorder-core/src/redaction-engine.ts) runs for `input` and `change` events on input, textarea, and select elements.

| Behavior              | Current implementation                                                                                                   |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Default text mode     | `masked`; preserves the first character and replaces the remainder with asterisks                                        |
| Sensitive fields      | Password and hidden types, built-in sensitive name patterns, and configured denylists generally produce an omitted value |
| Selector allowlist    | Allows raw values and can override sensitive-field omission                                                              |
| Checkboxes and radios | Record checked state; the early return bypasses sensitive-field omission                                                 |
| Hashed mode           | Uses a small deterministic, non-cryptographic hash; this is not encryption or anonymization                              |
| Value normalization   | Collapses whitespace, trims text, and limits captured values to 200 characters                                           |

The configuration also exposes selector denylists, field-name patterns, input-type denylists, and query-parameter redaction. The definitions are in [RecorderConfig](../packages/schema/src/index.ts), with defaults in [utils.ts](../packages/recorder-core/src/utils.ts).

Raw text capture is intended to require explicit permission in extension mode or an allowlisted selector. That restriction is not fully enforced: `textInputMode: 'raw'` can fall through to raw capture outside that intended gate. The existing value-capture test detects this failure.

## Gaps outside the value pipeline

| Capture path                        | Current limitation                                                                                                                     |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Keyboard events                     | `key` and `code` are recorded even for password fields, independently of input-value redaction                                         |
| Snapshot fragments                  | Sanitization visits descendant controls but skips a control at the fragment root; original text can also be appended as a comment      |
| Visible context and target metadata | Text, labels, and attributes do not pass through the input-value redaction engine                                                      |
| URLs                                | Event URL query filtering does not cover every URL field; session URLs, referrers, target links, and hashes can retain data            |
| Network settings                    | Custom filters and query-redaction settings can remain at defaults because the tracker is created before user configuration is applied |
| Identity storage                    | Initialization can write a visitor ID to local storage before a requested memory-only preference is applied                            |

The recorder also does not enforce the `capture.clicks`, `capture.inputs`, and `capture.keyboard` switches. They should not be used as exclusion controls until configuration handling is fixed.

## Storage and delivery

With a blank endpoint in `gtm` mode, the recorder uses a no-op transport or logs batches when debug is enabled. It still retains events and snapshots in memory and may persist identity. The extension stores per-tab recordings in `chrome.storage.local` and exposes export controls.

Network tracking does not read request or response bodies, but URLs and page context can themselves contain sensitive data. Keeping the HTTP endpoint blank avoids remote recorder delivery; it does not remove sensitive information from console output, local storage, or exports.

## Work required

1. Apply configuration before creating or starting capture subsystems.
2. Use consistent filtering for keyboard metadata, URLs, attributes, visible text, and snapshots.
3. Sanitize both fragment roots and descendants, including appended text.
4. Enforce raw-value mode restrictions and define allowlist/denylist precedence explicitly.
5. Verify full exported sessions and transport payloads with synthetic sensitive values, in addition to testing the value helper in isolation.

See [development status](development.md#verification-status) for the current checks and [architecture](architecture.md) for data flow.
