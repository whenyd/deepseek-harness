# Agent Note: tsdown config loading on runtimes without native TypeScript

Status: implemented

English | [中文](2026-09-22-tsdown-config-loader-runtime.zh.md)

## Problem

`pnpm run build` fails on a fresh clone wherever the Node runtime reports `process.features.typescript` as false, which some packaged Node 22 builds do, and the declared engine range `^22.19.0 || >=24.0.0` admits them. tsdown's `--config-loader auto` resolves to the `unrun` loader when neither Bun nor native TypeScript support is available, and tsdown declares `unrun` only as an optional peerDependency, so pnpm never installs it: the build stops at `Failed to import module "unrun"`.

Declaring the missing peer exposes a second failure. The unrun loader evaluates a compiled copy of each tsdown config, and the `import.meta.url` it substitutes there makes `new URL('../..', import.meta.url)` in [`tsdown.client.ts`](../../../../packages/client/tsdown.client.ts) resolve to `packages/` instead of the repository root. The preset's `workspaceManifest` then globs `packages/packages/*/*/package.json`, finds nothing, and the build fails with `tsdown: no packages/*/*/package.json declares the name @deepseek-ai/dsh-api-terminal-controller`. Runtimes where the auto loader picks the native path never import unrun and never see either failure.

## Decision

Root [`package.json`](../../../../package.json) declares `unrun` (`^0.3.1`) in `devDependencies` beside tsdown's other optional peers `tsx`, `publint`, and `typescript`, so the auto loader has a loader to resolve on every runtime in the engines range. It is development tooling only: no published artifact contains it, and [`THIRD_PARTY_NOTICES.md`](../../../../THIRD_PARTY_NOTICES.md) lists it as MIT development tooling.

[`tsdown.client.ts`](../../../../packages/client/tsdown.client.ts) locates the repository root by walking up from `process.cwd()` to the nearest `pnpm-workspace.yaml`. A workspace build evaluates package configs with the repository root as the working directory, and `pnpm --filter <pkg> bundle` runs with the package directory; both forms reach the root without depending on the config loader's `import.meta.url`.

`pnpm-lock.yaml` adds `unrun@0.3.1` and tsdown's peer suffix.

## Alternatives considered

**Require a Node runtime with native TypeScript support.** The engine floor is a version range, and `process.features.typescript` varies across builds of one version, so a version check cannot select a runtime that loads the configs.

**Pass `--config-loader tsx` at every tsdown call site.** tsdown accepts that value only through its CLI or programmatic API, not through the config file; the root scripts and every package-level `bundle`/`watch` script would need the flag, and a new call site could silently regress to the broken default.

**Keep the `import.meta.url` repository root and require a loader that preserves it.** The URL-relative climb is correct under the native and tsx loaders but wrong under unrun; the working-directory walk gives the same root under all three.

## Consequences

A fresh install reaches tsdown's configs on every runtime in the engines range, and the loader choice no longer rests on a feature compiled into the Node binary. The cost is one development-only dependency, a regenerated notices file and lockfile entry, and a preset that assumes its process starts inside the repository: a build launched elsewhere throws `tsdown: no pnpm-workspace.yaml above the working directory marks the repository root` instead of globbing the wrong directory.

Verification: `pnpm run build` completes on Node v22.22.1, where `process.features.typescript` is false and the loader is unrun; reverting the working-directory walk reproduces `tsdown: no packages/*/*/package.json declares the name @deepseek-ai/dsh-api-terminal-controller`; `pnpm run verify-third-party-notices` and a frozen-lockfile install pass.

## Related

- [TSC-first build and one compiler ownership](2026-06-17-ts-build-config.md) owns the build phases and tsdown's bundling responsibility; this note adds only how tsdown loads its configs.
- [Bundle the Desktop main process after the workspace tsdown pass](2026-09-22-desktop-main-bundle-after-workspace-tsdown.md) owns the root workspace build list and Desktop bundling order.
