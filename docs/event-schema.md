# Event model

[← Repository overview](../README.md)

The canonical contracts live in [packages/schema/src/index.ts](../packages/schema/src/index.ts). Both runtime modes use the same types. The exported JSON Schema describes part of the batch envelope; it is not a complete runtime validator for all nested fields.

## Batches and session exports

| Shape             | Contents                                                                                      | Used by                                                             |
| ----------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `TransportBatch`  | Batch ID, mode, send timestamp, session metadata, and events; optional app ID and endpoint    | HTTP, console, and extension bridge transports                      |
| `ExportedSession` | Export timestamp, mode, session metadata, diagnostics, snapshots, and events; optional app ID | `window.FlowRecorder.exportSession()` and extension status messages |

A transport batch does **not** contain full snapshots. An event's `snapshot_ref` points to an object held in the session export. Consumers that need HTML fragments must obtain that export; the HTTP path does not deliver the snapshot objects today.

## Event envelope

| Field group | Examples                                                                                | Meaning                                                                |
| ----------- | --------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Identity    | `event_id`, `visitor_id`, `session_id`, `pageview_id`, `route_id`, `tab_id`, `state_id` | Correlates interactions with a visitor, page, route, and current state |
| Ordering    | `sequence_no`, `ts_unix_ms`, `ts_perf_ms`                                               | Recorder-local order, wall-clock time, and performance-clock time      |
| Source      | `mode`, `category`, `event_type`                                                        | Runtime mode and event classification                                  |
| Page        | `url`, `url_path`, `url_hash`, `title`, `referrer`                                      | Page location and document metadata                                    |
| Environment | `viewport`, `document_ready_state`, `visibility_state`                                  | Viewport geometry, scroll position, and document state                 |

Some identifiers and optional context fields can be null. A state ID is not available until a state has settled. Sequence numbers belong to the recorder instance and do not establish ordering across tabs.

## Event categories

| Category     | Examples                                                                                                                                |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------------- |
| `user`       | `click`, `input`, `change`, `submit`, keyboard events, `scroll.start`, `scroll.progress`, `scroll.end`                                  |
| `navigation` | `document.load`, `history.pushState`, `history.popstate`, `route.change`, visibility and page lifecycle events                          |
| `system`     | `network.request.start`, `network.request.end`, `dom.mutation.burst`, `state.settling.start`, `state.settled`, `state.snapshot.created` |

Raw browser events remain in the stream alongside higher-level markers. One interaction can produce several events, such as pointer, mouse, click, network, and settlement records. A replay generator will need to choose which represent executable actions.

## Context for future replay

- `action_kind` normalizes events into categories such as `action.click`, `action.type`, and `action.submit`.
- `target` describes the element, its geometry, and nearby form, landmark, heading, or container.
- `selectors` contains ranked locator candidates with strategy, confidence, stability, and rationale. Ranking does not guarantee uniqueness.
- `visible_context` summarizes selected UI elements, headings, focus, and detected dialogs or drawers.
- `captured_value` and `redaction` describe input-value handling. They do not certify that other event fields are redacted.
- `replay_hints` carries suggested waits and context flags. It is advisory metadata, not executable automation.
- `network` carries request metadata when captured; it does not include request or response bodies.

## State transitions and snapshots

The recorder emits `state.settled` with the newly created `state_id`, `state_id_after`, and a settlement reason. If snapshot capture is enabled, the event also references the snapshot. Earlier user events are not updated with that resulting state.

| Snapshot mode    | Behavior                                                              |
| ---------------- | --------------------------------------------------------------------- |
| `off`            | No state snapshots                                                    |
| `balanced`       | Bounded fragments of the latest target and selected nearby containers |
| `enhanced-local` | The same target context plus a truncated body fragment                |

Snapshots include visible context, a DOM signature, route-template hints, and HTML fragments. Fragment limits are based on string length rather than encoded byte size. A snapshot can have no fragments when there is no eligible target.

## Example data

The [session fixtures](../examples/exported-sessions) cover [forms](../examples/exported-sessions/simple-form-flow.json), [modals](../examples/exported-sessions/modal-flow.json), [SPA navigation](../examples/exported-sessions/spa-navigation-flow.json), and [async lists](../examples/exported-sessions/async-loaded-list-flow.json).

These are illustrative fixtures, not golden recordings of the current implementation. Some show enriched action-to-state relationships that the runtime does not currently backfill. Use the TypeScript contracts and a fresh demo recording to check current behavior.

For data handling limitations, see [privacy and redaction](privacy-redaction.md).
