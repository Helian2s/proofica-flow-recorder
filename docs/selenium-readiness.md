# Replay roadmap

[← Repository overview](../README.md)

Flow Recorder currently captures data that could support a Selenium generator. It does not generate test files, execute Selenium, or verify that a recorded session can be replayed.

## Foundations already present

| Captured information                       | Possible use in a generator                                                       |
| ------------------------------------------ | --------------------------------------------------------------------------------- |
| Ranked locator candidates                  | Try test hooks and semantic locators before structural fallbacks                  |
| Normalized `action_kind`                   | Map input, click, select, toggle, submit, and scroll activity to automation steps |
| Route and state markers                    | Segment a flow and identify changes after an action                               |
| DOM-quiet and network-idle hints           | Suggest waits around asynchronous interactions                                    |
| Form, modal, landmark, and heading context | Scope locators and identify the active view                                       |
| Snapshot fragments and visible context     | Inspect surrounding UI when selecting locators or assertions                      |

These are inputs to a future generator, not guarantees of replay correctness. The [example sessions](../examples/exported-sessions) illustrate possible annotations, including relationships that the current recorder does not populate on earlier action events.

## Work before generation

1. **Make capture dependable.** Apply configuration before initialization, close redaction gaps, and restore passing build and verification commands.
2. **Preserve complete recordings.** Deliver snapshot bodies with their references, refresh extension exports before download, and add retry and retention policies.
3. **Define action-to-state relationships.** Associate actions with later settled states explicitly, including timeout outcomes, navigation, and overlapping requests.
4. **Validate locator selection.** Check uniqueness, escaping, repeated elements, dynamic IDs, frames, and shadow roots.
5. **Define replay inputs.** Supply synthetic values or named test parameters when recorded values are masked or omitted; exported values may also be normalized or truncated.

## A first generator

A small initial generator could consume an `ExportedSession`, select executable actions from the event stream, map locator strategies to Selenium calls, and insert explicit waits. It should report unsupported actions and ambiguous targets rather than silently guessing.

A useful first milestone would replay the local demo's form submission and one delayed modal flow. Validation should compare the resulting UI state and surface failures caused by changed locators or timing.

After that, the project could explore reusable page abstractions, richer assertions, and clustering similar settled states. These remain future work.
