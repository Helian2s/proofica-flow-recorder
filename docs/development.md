# Development

[← Repository overview](../README.md)

This guide covers local setup, repository commands, and the current verification baseline.

## Prerequisites

- Node.js. The repository does not currently declare an `engines` range.
- pnpm **9.12.0**, as declared in [package.json](../package.json).
- Chrome for the unpacked extension.
- Playwright's Chromium binary for the browser smoke test; install it with the command below.

There is no committed `pnpm-lock.yaml`, so dependency resolution is not reproducible across install dates. A lockfile and a documented Node version are part of the remaining build work.

## Run the demo

From the repository root:

```bash
pnpm install
pnpm dev:demo
```

Open [localhost:4173](http://127.0.0.1:4173) and follow the [demo walkthrough](../README.md#try-it-locally). The development script bundles source directly with esbuild and serves it locally. It watches source files, but browser refreshes are manual.

You can set the host and port explicitly:

```bash
pnpm dev:demo --host 127.0.0.1 --port 4173
```

For the extension, run `pnpm dev:extension` in another terminal and follow the [loading instructions](extension-local-mode.md#load-locally). Both commands remain running until stopped with Ctrl+C.

## Commands

| Command              | Purpose                                                                    |
| -------------------- | -------------------------------------------------------------------------- |
| `pnpm dev:demo`      | Bundle/watch the demo and start its HTTP server                            |
| `pnpm dev:extension` | Bundle/watch the extension and copy its static files                       |
| `pnpm build`         | Bundle all packages, SDK IIFE, extension, and demo; then emit declarations |
| `pnpm test:unit`     | Run Vitest                                                                 |
| `pnpm test:e2e`      | Run the Playwright demo smoke test                                         |
| `pnpm test`          | Run Vitest, then Playwright only if Vitest succeeds                        |
| `pnpm lint`          | Run ESLint across TypeScript and JavaScript files                          |
| `pnpm format`        | Rewrite repository files with Prettier                                     |

Generated application and package bundles live in their respective `dist` directories. Those directories are ignored by Git. The release helper in `scripts/release.ts` only prints a checklist; it does not publish artifacts.

## Browser test setup

```bash
pnpm exec playwright install chromium
pnpm test:e2e
```

Playwright starts the demo server at `127.0.0.1:4173`, or reuses an existing server outside CI. Its current test visits the forms route and checks that route, selector, and submit context data are exported. It does not test the Chrome extension.

If startup fails because Chromium is missing, install the binary first. A missing browser is an environment issue and does not establish whether the application test passes.

## Verification status

The local review on **2026-09-10** used Node.js 24.19.0 and the available pnpm 11.19.0 runtime with freshly resolved dependencies, including TypeScript 5.9.3 and Vitest 2.1.9. This is an observed baseline, not a supported-version matrix or CI result; pnpm's automatic dependency reinstallation was disabled for the checks.

| Check                            | Observed result                                                                                                                         |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Workspace build                  | JavaScript bundles generated; declaration emission failed with `TS6059` because workspace source imports fall outside package `rootDir` |
| Additional core type check       | Also reported strict optional-property and browser/Node timer-type errors                                                               |
| Vitest after JavaScript bundling | 7 tests passed and 2 failed; a further suite failed because Vitest collected the Playwright spec                                        |
| ESLint                           | 10 errors                                                                                                                               |
| Playwright                       | Test could not launch because the required Chromium binary was absent; browser behavior was not verified                                |

The two failing unit tests concern raw-value mode restrictions and scroll progress. The scroll test advances beyond its idle timeout between events, so its expected progress marker needs review alongside the implementation. The raw-value test exposes an actual mode-gating failure.

The [Vitest include patterns](../vitest.config.ts) collect `apps/*/test/**/*.spec.ts`, including the Playwright file. On a clean checkout before bundling, SDK tests can also fail to resolve package entry points in `dist`. These issues mean `pnpm test` is not currently a passing end-to-end verification command.

## Useful places to work

- [Recorder lifecycle](../packages/recorder-core/src/index.ts): configuration, subsystem creation, event assembly, and retained session state.
- [Unit tests](../packages/recorder-core/test): focused subsystem behavior.
- [Browser smoke test](../apps/demo-spa/test/recorder.spec.ts): the existing demo flow.
- [Build script](../scripts/build-all.ts) and [base TypeScript config](../tsconfig.base.json): bundling and workspace resolution.
- [Privacy notes](privacy-redaction.md): gaps that need checks against complete exported payloads.
- [Replay roadmap](selenium-readiness.md): work planned after recording reliability.
