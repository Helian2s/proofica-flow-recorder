# Browser SDK and GTM integration

[← Repository overview](../README.md)

The SDK exposes `window.FlowRecorder` for script-based installation. The `gtm` mode is a first-party integration path; it does not require Google Tag Manager to run. This guide covers a local prototype setup. Review the [privacy limitations](privacy-redaction.md) before choosing a test page.

## Bundle outputs

| Format       | Generated path                                         |
| ------------ | ------------------------------------------------------ |
| Browser IIFE | `packages/sdk-browser/dist/iife/flow-recorder.iife.js` |
| ESM          | `packages/sdk-browser/dist/index.js`                   |
| CommonJS     | `packages/sdk-browser/dist/index.cjs`                  |

`pnpm build` writes these JavaScript bundles before attempting declarations. The full command currently exits with a declaration error, so the presence of a bundle does not mean the workspace build passed. The [local demo](../README.md#try-it-locally) has its own development bundling command.

There is no published CDN URL or receiving endpoint included in the repository.

## Script installation

Serve the generated IIFE bundle from a static location you control. Load it before calling the API:

```html
<!-- Replace this example URL with the location of your generated bundle. -->
<script src="https://static.example.com/flow-recorder.iife.js"></script>
<script>
  window.FlowRecorder.init({
    endpoint: '',
    appId: 'demo-app',
    mode: 'gtm',
    autoStart: true,
    debug: true,
  });
</script>
```

With this configuration, batches go to the console and the session remains available through `exportSession()`. Setting `debug: false` with a blank endpoint selects the no-op transport; recording and in-memory exports still occur.

For a GTM experiment, use the same loading and initialization sequence in a Custom HTML tag. The URL above is a placeholder, not a hosted artifact. An [example snippet](../examples/gtm-custom-html/snippet.html) is also included. No GTM Custom Template is supplied.

Load the bundle once per page instance. For SPA navigation, the existing recorder observes History API changes; it does not need to be reinstalled on every route. Avoid running the SDK and extension recorder simultaneously in the same document.

## Browser API

| Method            | Behavior                                                                                    |
| ----------------- | ------------------------------------------------------------------------------------------- |
| `init(config)`    | Creates/configures the recorder and starts by default; returns the API                      |
| `start(config?)`  | Starts recording, optionally merging configuration                                          |
| `stop()`          | Detaches capture and initiates a flush; retains history for export                          |
| `flush(reason?)`  | Requests queue delivery and returns a promise; transport errors are reported in diagnostics |
| `exportSession()` | Returns the current in-memory session or null before initialization                         |
| `getStatus()`     | Returns started state, mode, URL, event count, queue size, and identifiers                  |
| `version`         | SDK version string                                                                          |

For manual start:

```js
window.FlowRecorder.init({
  endpoint: '',
  appId: 'local-experiment',
  mode: 'gtm',
  autoStart: false,
  debug: true,
});

window.FlowRecorder.start();
// Interact with the page, then inspect the captured data.
console.log(window.FlowRecorder.exportSession());
```

`autoStart: false` delays capture listeners, but initialization still constructs identity state. It is not a guarantee of zero storage writes. Other [configuration gaps](architecture.md#implementation-gaps) also apply.

## HTTP delivery

A non-empty endpoint selects `FetchTransport`, which posts a JSON `TransportBatch` with `keepalive: true`. The caller must provide a compatible receiving service. Snapshot bodies are not included in those batches.

The current `sendBeacon` path sends a flush marker containing the reason and timestamp; it does not send queued event data. Failed sends are requeued without automatic retry scheduling. See the [event model](event-schema.md) and [transport implementation](../packages/transport/src/index.ts) before building a consumer.
