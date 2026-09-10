# Chrome extension

[← Repository overview](../README.md)

The Manifest V3 extension launches the shared recorder in a tab's page context and stores recording data locally. It is a development harness for this portfolio project. Start with the included demo and synthetic data.

## Load locally

1. Install dependencies from the repository root with `pnpm install`.
2. Run `pnpm dev:demo` and open [localhost:4173](http://127.0.0.1:4173).
3. In another terminal, run `pnpm dev:extension`.
4. Open `chrome://extensions` and enable **Developer mode**.
5. Choose **Load unpacked** and select `apps/extension/dist` inside your checkout.
6. Keep the demo's SDK recorder stopped. Open the extension popup on the demo tab and click **Start recording**.

The extension development command writes and watches its own bundles. A successful full workspace build is not required for this path. After rebuilding, reload the extension in Chrome and reload the demo page to replace already-injected code.

## Components

| Component                                                   | Responsibility                                                                 |
| ----------------------------------------------------------- | ------------------------------------------------------------------------------ |
| [Service worker](../apps/extension/src/background/index.ts) | Receives popup commands, injects content code, and stores per-tab session data |
| [Content script](../apps/extension/src/content/index.ts)    | Loads the page bridge and forwards commands, batches, and status messages      |
| [Page bridge](../apps/extension/src/page-bridge/index.ts)   | Runs the browser API in `extension-local` mode                                 |
| [Popup](../apps/extension/src/popup/index.ts)               | Displays cached status and provides recording/export controls                  |
| [Options](../apps/extension/src/options/index.ts)           | Persists raw-capture and debug preferences                                     |

The [manifest](../apps/extension/manifest.json) requests `activeTab`, `scripting`, `storage`, and `tabs`. The page bridge is declared as a web-accessible resource. Recording uses the same core as SDK mode; there is no separate extension capture engine.

## Popup controls

| Control                    | Current behavior                                                                                                           |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **Start recording**        | Loads configuration from extension preferences and starts the page recorder                                                |
| **Stop recording**         | Stops capture and requests a status update through the page bridge                                                         |
| **Flush and export**       | Downloads the cached session export; despite the label, the button does not send a flush command                           |
| **Clear data**             | Resets stored tab data and replaces the page API; capture can restart if the previous configuration had auto-start enabled |
| **Allow raw text capture** | Saves a preference read by the next start command; defaults to off                                                         |
| **Debug logging**          | Saves a preference read by the next start command                                                                          |

Saving a toggle does not immediately reconfigure an active recording. Privacy filtering has [known gaps](privacy-redaction.md), including paths outside text-value capture.

## Export behavior

Batches update the service worker's stored event list. Full exported sessions arrive separately in status messages, which the page bridge emits at startup and after commands. The popup reads that cached export without first obtaining a fresh one.

Consequently, a download can omit events even when the stored event count has increased. Stopping sends an updated status, but message handling and storage are asynchronous, so an immediate export is not guaranteed to include it.

For inspecting current data during development, select the tab's page context in DevTools and read the page API directly after the interaction settles:

```js
window.FlowRecorder.getStatus();
window.FlowRecorder.exportSession();
```

This returns the page recorder's in-memory data. It does not fix the popup's cached-export behavior or wait for pending UI work.

## Scope and limitations

- Injection is triggered by popup commands. There is no automatic reinjection handler for full-page navigation.
- The bridge replaces `window.FlowRecorder`; avoid starting both runtime modes on the same page.
- Storage contains per-tab data, but retention and cleanup are incomplete.
- Status and export controls do not yet provide reliable acknowledgments of completed page commands and storage writes.
- Bridge message checks identify source/type fields but do not fully validate nested payloads.

See [architecture](architecture.md#chrome-extension) for the message sequence and [development](development.md) for verification commands.
