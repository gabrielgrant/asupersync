# Asupersync Browser Edition — JavaScript/TypeScript API Reference

This document is the end-user reference for writing async browser code with
**Asupersync Browser Edition**. It covers installation, core concepts, every
public API with type signatures, and usage examples for vanilla JS/TS, React,
and Next.js. All information is derived directly from the source code; treat
it as ground truth.

---

## Table of Contents

1. [Overview](#overview)
2. [Requirements & Browser Support](#requirements--browser-support)
3. [Package Hierarchy](#package-hierarchy)
4. [Installation](#installation)
5. [Quick Start](#quick-start)
6. [Core Concepts](#core-concepts)
   - [The Outcome Type](#the-outcome-type)
   - [Handles](#handles)
   - [Structured Concurrency: Runtimes and Regions](#structured-concurrency-runtimes-and-regions)
   - [Tasks](#tasks)
   - [Cancellation](#cancellation)
   - [Budgets](#budgets)
7. [`@asupersync/browser` — High-Level SDK](#asupersyncbrowser--high-level-sdk)
   - [Initialization](#initialization)
   - [BrowserRuntime](#browserruntime)
   - [RegionHandle](#regionhandle)
   - [TaskHandle](#taskhandle)
   - [FetchHandle](#fetchhandle)
   - [CancellationToken](#cancellationtoken)
   - [Utility Functions](#utility-functions)
   - [Support Detection](#support-detection)
8. [`@asupersync/browser-core` — Low-Level Bindings](#asupersyncbrowser-core--low-level-bindings)
   - [Initialization](#initialization-1)
   - [ABI Functions](#abi-functions)
   - [Handle Classes](#handle-classes)
   - [Constants and Factories](#constants-and-factories)
   - [Raw WASM Bindings](#raw-wasm-bindings)
9. [TypeScript Types Reference](#typescript-types-reference)
10. [Fetch API](#fetch-api)
11. [WebSocket API](#websocket-api)
12. [React Integration (`@asupersync/react`)](#react-integration-asupersyncreact)
13. [Next.js Integration (`@asupersync/next`)](#nextjs-integration-asupersyncnext)
14. [Error Handling](#error-handling)
15. [ABI Versioning](#abi-versioning)

---

## Overview

Asupersync Browser Edition packages the Asupersync async runtime as a WASM
module with a thin JS/TS wrapper. It brings structured concurrency, explicit
cancellation, and capability-scoped effects to the browser, replacing ad-hoc
`Promise.race`, fire-and-forget `async/await`, and implicit `fetch` calls with
a principled, leak-free model.

Key properties:

- **Structured concurrency** — every task is owned by exactly one region. A
  region cannot close until all its tasks have finished or been drained.
- **Explicit cancellation** — cancellation is a three-phase protocol
  (`requested → draining → finalizing → completed`), not a silent drop.
- **Capability security** — effects like `fetch` and WebSocket are requested
  through explicit handles tied to a region's lifetime, not ambient globals.
- **Four-valued outcomes** — results carry `ok`, `err`, `cancelled`, or
  `panicked` tags, making every non-success case explicit and handleable.

---

## Requirements & Browser Support

The runtime requires a browser that provides:

| Feature | Required |
|---------|----------|
| `WebAssembly` | ✅ |
| `fetch` | ✅ |
| `AbortController` | ✅ |
| `WebSocket` | ✅ |
| `window` and `document` | ✅ |

These correspond to all evergreen browsers (Chrome ≥ 79, Edge ≥ 79,
Firefox ≥ 72, Safari ≥ 15) released after 2020. Older browsers that lack
`WebAssembly.instantiateStreaming` or `AbortController` are not supported.
**Node.js server and edge runtimes are not supported for direct runtime
creation** — use the bridge adapters from `@asupersync/next` instead.

Use [`detectBrowserRuntimeSupport()`](#support-detection) at startup to get
a structured diagnostics report.

---

## Package Hierarchy

Four npm packages are layered from low-level to high-level:

```
@asupersync/browser-core   ← raw WASM bindings + ABI types
        ↑
@asupersync/browser        ← ergonomic browser SDK (recommended starting point)
        ↑
@asupersync/react          ← React StrictMode-safe provider + hooks
        ↑
@asupersync/next           ← Next.js App Router client/server/edge adapters
```

Start at the highest layer that covers your framework. Only drop down to
`browser-core` when you need raw ABI handles, custom metadata-driven
compatibility checks, or low-level WASM introspection.

---

## Installation

```bash
# Vanilla JS/TS or any bundler
npm install @asupersync/browser

# React apps
npm install @asupersync/react react react-dom

# Next.js apps
npm install @asupersync/next next react react-dom
```

`@asupersync/browser-core` is automatically installed as a dependency. You
only need to install it explicitly if your project needs raw ABI access.

---

## Quick Start

### Vanilla TypeScript

```ts
import { createBrowserRuntime, unwrapOutcome } from "@asupersync/browser";

// 1. Create the runtime — loads and initializes the WASM module.
const runtimeOutcome = await createBrowserRuntime();
const runtime = unwrapOutcome(runtimeOutcome);

// 2. Open a structured-concurrency region.
const scopeOutcome = runtime.enterScope("my-app");
const scope = unwrapOutcome(scopeOutcome);

// 3. Issue a tracked HTTP request within that region.
const fetchOutcome = scope.fetchRequest({ url: "/api/data", method: "GET" });
if (fetchOutcome.outcome !== "ok") {
  console.error("fetch failed:", fetchOutcome);
} else {
  const fetchHandle = fetchOutcome.value;
  // fetchHandle is a FetchHandle — use it to track lifecycle events.
}

// 4. Always close the scope when done. The runtime waits for quiescence.
scope.close();
runtime.close();
```

### Using `withScope` for automatic cleanup

```ts
import { createBrowserRuntime } from "@asupersync/browser";

const runtimeOutcome = await createBrowserRuntime();
if (runtimeOutcome.outcome !== "ok") {
  console.error("Runtime failed to start:", runtimeOutcome);
  return;
}
const runtime = runtimeOutcome.value;

const result = await runtime.withScope(async (scope) => {
  const fetchOutcome = scope.fetchRequest({
    url: "/api/items",
    method: "GET",
  });
  if (fetchOutcome.outcome !== "ok") {
    return fetchOutcome; // propagate the non-ok outcome
  }
  // ... do work with fetchOutcome.value ...
  return { outcome: "ok" as const, value: "done" };
});

runtime.close();
```

---

## Core Concepts

### The Outcome Type

Every operation returns an `Outcome<T, E>` instead of throwing or returning
`null`. There are four variants:

```ts
type Outcome<T = WasmValue, E = AbiFailure> =
  | { outcome: "ok";        value: T }
  | { outcome: "err";       failure: E }
  | { outcome: "cancelled"; cancellation: AbiCancellation }
  | { outcome: "panicked";  message: string };
```

Always pattern-match the `outcome` field before using the value:

```ts
const result = runtime.enterScope("work");
switch (result.outcome) {
  case "ok":
    const scope = result.value;
    break;
  case "err":
    console.error(result.failure.code, result.failure.message);
    break;
  case "cancelled":
    console.warn("cancelled:", result.cancellation.kind);
    break;
  case "panicked":
    console.error("panic:", result.message);
    break;
}
```

`unwrapOutcome(outcome)` is a convenience helper that returns the value for
`"ok"` outcomes and throws an `Error` for any other variant:

```ts
import { unwrapOutcome } from "@asupersync/browser";
const scope = unwrapOutcome(runtime.enterScope("work")); // throws on non-ok
```

The `Outcome` factory object creates outcome envelopes manually:

```ts
import { Outcome } from "@asupersync/browser-core";
Outcome.ok(42);
Outcome.err("internal_failure", "unknown", "something went wrong");
Outcome.cancelled({ kind: "user", phase: "requested", ... });
Outcome.panicked("unexpected panic message");
```

### Handles

Handles are **opaque references** to live WASM-side objects. They carry three
fields: `kind`, `slot`, and `generation`. You never create handles manually —
they are returned by runtime operations and must be closed or joined to avoid
resource leaks.

```ts
interface HandleRef {
  kind: HandleKind;  // "runtime" | "region" | "task" | "cancel_token" | "fetch_request"
  slot: number;
  generation: number;
}
```

All handle classes (`RuntimeHandle`, `RegionHandle`, `TaskHandle`,
`CancellationToken`, `FetchHandle`) extend `BaseHandle`, which serializes to
a `HandleRef` via `.toJSON()`.

### Structured Concurrency: Runtimes and Regions

The runtime is the root context. Each region (scope) is a sub-tree that owns
zero or more tasks and can nest further child regions. When you close a region,
the runtime enforces quiescence: all tasks must have completed, been cancelled,
or been fully drained before the close returns.

```
BrowserRuntime
  └── RegionHandle (scope A)
        ├── TaskHandle
        └── RegionHandle (nested scope B)
              └── TaskHandle
```

Creating a region:

```ts
// From a runtime:
const scope = unwrapOutcome(runtime.enterScope("label"));

// Or from an existing scope (nested):
const child = unwrapOutcome(scope.enterScope("child"));
```

Always close scopes when done:

```ts
child.close();
scope.close();
runtime.close();
```

### Tasks

A task represents a unit of async work tracked by the runtime. You spawn one
into a region and can join it with an outcome or cancel it:

```ts
// Spawn a task placeholder in a scope:
const taskOutcome = scope.spawnTask({ label: "my-task" });
const task = unwrapOutcome(taskOutcome);

// Join the task with a result:
const joinResult = task.join({ outcome: "ok", value: undefined });

// Cancel the task:
const cancelResult = task.cancel("user", "user clicked cancel");
```

### Cancellation

Cancellation is a first-class three-phase protocol:

```
requested → draining → finalizing → completed
```

Each `CancellationToken` carries a `kind` string (e.g. `"user"`, `"timeout"`)
and an optional `message`. To cancel a task:

```ts
import { CancellationToken } from "@asupersync/browser";

const token = CancellationToken.user("user clicked stop");
const result = token.cancel(task);
// or
const result = task.cancel("timeout", "deadline exceeded");
```

When a task is cancelled the outcome's `cancellation` field includes the kind,
the phase at which it was captured, the originating region/task, and a
timestamp in nanoseconds.

### Budgets

A `Budget` constrains a task's scheduling behaviour:

```ts
interface Budget {
  pollQuota:    number;  // poll steps before yielding  (1 – 1,000,000)
  deadlineMs:   number;  // wall-clock deadline in ms   (0 – 86,400,000)
  priority:     number;  // scheduler priority          (0 – 255)
  cleanupQuota: number;  // cleanup poll steps          (0 – 1,000,000)
}
```

Use `createBudget()` to build a validated budget with safe defaults
(`pollQuota=1024`, `deadlineMs=30000`, `priority=100`, `cleanupQuota=256`):

```ts
import { createBudget } from "@asupersync/browser-core";

const budget = createBudget({ priority: 200, deadlineMs: 5000 });
```

`BUDGET_BOUNDS` exposes the validated ranges for each field:

```ts
import { BUDGET_BOUNDS } from "@asupersync/browser-core";
// BUDGET_BOUNDS.pollQuota.min === 1, .max === 1_000_000
```

---

## `@asupersync/browser` — High-Level SDK

This is the recommended package for browser applications without a specific
framework adapter.

### Initialization

#### `createBrowserRuntime(options?)`

```ts
async function createBrowserRuntime(
  options?: BrowserRuntimeOptions,
): Promise<Outcome<BrowserRuntime, AbiFailure>>;

interface BrowserRuntimeOptions {
  wasmInput?:       InitInput;          // URL/Response/buffer for the .wasm file
  consumerVersion?: AbiVersion | null;  // lock to a specific ABI version
  eagerInit?:       boolean;            // false to skip automatic WASM load (default true)
}
```

Loads the WASM module (if `eagerInit` is not `false`, which is the default when the option is omitted) and creates a root runtime handle. Returns an `Outcome<BrowserRuntime>`.

```ts
const runtimeOutcome = await createBrowserRuntime({
  wasmInput: "/static/asupersync_bg.wasm",
});
if (runtimeOutcome.outcome !== "ok") {
  throw new Error("Could not start runtime");
}
const runtime = runtimeOutcome.value;
```

#### `createBrowserScope(options?)`

A convenience wrapper that creates both a runtime and an initial scope in one
call. The returned `RegionHandle` is backed by the internally created runtime.
Close the scope through the handle when you are done.

```ts
async function createBrowserScope(
  options?: BrowserRuntimeOptions & BrowserScopeOptions,
): Promise<Outcome<RegionHandle, AbiFailure>>;
```

### BrowserRuntime

The `BrowserRuntime` class wraps a `CoreRuntimeHandle` from `browser-core`
and keeps track of the `consumerVersion` for all subsequent calls.

```ts
class BrowserRuntime {
  readonly core:            CoreRuntimeHandle;
  readonly consumerVersion: AbiVersion | null;
  readonly diagnostics:     BrowserSdkDiagnostics;

  close(consumerVersion?: AbiVersion | null): Outcome<void>;

  enterScope(
    label?: string,
    consumerVersion?: AbiVersion | null,
  ): Outcome<RegionHandle>;

  withScope<T>(
    fn: (scope: RegionHandle) => Promise<Outcome<T>> | Outcome<T>,
    options?: BrowserScopeOptions,
  ): Promise<Outcome<T>>;

  toJSON(): HandleRef;
}
```

`withScope` opens a region, calls `fn`, and always closes the region in the
`finally` block, even if `fn` throws:

```ts
const result = await runtime.withScope(async (scope) => {
  const fetch = unwrapOutcome(scope.fetchRequest({ url: "/api", method: "GET" }));
  return { outcome: "ok" as const, value: fetch };
}, { label: "api-call" });
```

### RegionHandle

```ts
class RegionHandle {
  readonly core:            CoreRegionHandle;
  readonly consumerVersion: AbiVersion | null;
  readonly runtime:         BrowserRuntime | null;

  close(consumerVersion?: AbiVersion | null): Outcome<void>;

  enterScope(
    label?: string,
    consumerVersion?: AbiVersion | null,
  ): Outcome<RegionHandle>;

  spawnTask(
    options?: Omit<TaskSpawnRequest, "scope">,
    consumerVersion?: AbiVersion | null,
  ): Outcome<TaskHandle>;

  fetchRequest(
    options: Omit<FetchRequest, "scope">,
    consumerVersion?: AbiVersion | null,
  ): Outcome<FetchHandle>;

  openWebSocket(
    url: string,
    protocols?: string[],
    consumerVersion?: AbiVersion | null,
  ): Outcome<TaskHandle>;

  toJSON(): HandleRef;
}
```

Example — nested scopes:

```ts
const outer = unwrapOutcome(runtime.enterScope("outer"));
const inner = unwrapOutcome(outer.enterScope("inner"));
inner.close(); // inner must close first
outer.close();
```

### TaskHandle

```ts
class TaskHandle {
  readonly core:            CoreTaskHandle;
  readonly consumerVersion: AbiVersion | null;

  join(
    outcome: Outcome,
    consumerVersion?: AbiVersion | null,
  ): Outcome<WasmValue>;

  cancel(
    tokenOrKind: CancellationToken | string,
    message?: string,
    consumerVersion?: AbiVersion | null,
  ): Outcome<void>;

  toJSON(): HandleRef;
}
```

`join` registers the outcome of the async work the task represents. Pass the
result of your logic as an `Outcome` envelope:

```ts
const task = unwrapOutcome(scope.spawnTask({ label: "compute" }));

// Simulate completion
const joinResult = task.join({ outcome: "ok", value: "result" });
```

### FetchHandle

A `FetchHandle` is an opaque reference to a tracked HTTP request that was
issued inside a region. It inherits the region's lifetime, so the request is
automatically cancelled when the region closes.

```ts
class FetchHandle {
  readonly core:            CoreFetchHandle;
  readonly consumerVersion: AbiVersion | null;
  toJSON(): HandleRef;
}
```

### CancellationToken

`CancellationToken` in `@asupersync/browser` is a **value object** (not a
WASM-side handle) that carries a `kind` string and optional `message`. It
is not the same as `CancellationToken` in `browser-core`, which is an opaque
handle class for WASM-side cancel tokens.

```ts
class CancellationToken {
  readonly kind:            string;
  readonly message?:        string;
  readonly consumerVersion: AbiVersion | null;

  constructor(
    kindOrOptions: string | CancellationTokenOptions,
    message?: string,
    consumerVersion?: AbiVersion | null,
  );

  // Static factories
  static user(message?: string, consumerVersion?: AbiVersion | null): CancellationToken;
  static timeout(message?: string, consumerVersion?: AbiVersion | null): CancellationToken;

  // Return a copy with a different message
  withMessage(message?: string): CancellationToken;

  // Cancel a task with this token's kind/message
  cancel(
    task: TaskHandle | CoreTaskHandle | HandleRef,
    consumerVersion?: AbiVersion | null,
  ): Outcome<void>;

  // Create an ABI cancellation envelope for testing / diagnostics
  toCancellation(phase?: AbiCancellation["phase"]): AbiCancellation;
}
```

Usage:

```ts
const userCancel  = CancellationToken.user("Stopped by user");
const timeoutCancel = CancellationToken.timeout("30 s deadline exceeded");

// Later:
userCancel.cancel(task);
// or
task.cancel(userCancel);
// or
task.cancel("user", "Stopped by user");
```

`createCancellationToken` is a function alias for `new CancellationToken(...)`:

```ts
import { createCancellationToken } from "@asupersync/browser";
const token = createCancellationToken("user", "manually cancelled");
```

### Utility Functions

#### `unwrapOutcome<T>(outcome): T`

Returns the `value` from an `"ok"` outcome, or throws a descriptive `Error`
for any other variant.

#### `formatOutcomeFailure(outcome): string`

Returns a human-readable string for any non-ok outcome:

```ts
import { formatOutcomeFailure } from "@asupersync/browser";

const msg = formatOutcomeFailure({ outcome: "err", failure: { code: "invalid_handle", recoverability: "permanent", message: "handle expired" }});
// → "invalid_handle: handle expired"
```

#### `createBrowserSdkDiagnostics(consumerVersion?): BrowserSdkDiagnostics`

Returns runtime diagnostics including the live ABI version and fingerprint:

```ts
interface BrowserSdkDiagnostics {
  abiVersion:      AbiVersion;
  abiFingerprint:  number;
  abiMetadata:     BrowserAbiMetadata;
  consumerVersion: AbiVersion | null;
}
```

### Support Detection

#### `detectBrowserRuntimeSupport(globalObject?): BrowserRuntimeSupportDiagnostics`

Checks whether the current environment can run Browser Edition without
throwing. Returns a structured report:

```ts
interface BrowserRuntimeSupportDiagnostics {
  supported:    boolean;
  packageName:  "@asupersync/browser";
  reason:       "missing_global_this" | "missing_browser_dom" | "missing_webassembly" | "supported";
  message:      string;
  guidance:     string[];
  capabilities: BrowserCapabilitySnapshot;
}

interface BrowserCapabilitySnapshot {
  hasAbortController: boolean;
  hasDocument:        boolean;
  hasFetch:           boolean;
  hasWebAssembly:     boolean;
  hasWebSocket:       boolean;
  hasWindow:          boolean;
}
```

```ts
import { detectBrowserRuntimeSupport } from "@asupersync/browser";

const diag = detectBrowserRuntimeSupport();
if (!diag.supported) {
  console.warn(diag.message, diag.guidance);
}
```

#### `assertBrowserRuntimeSupport(diagnostics?): BrowserRuntimeSupportDiagnostics`

Like `detectBrowserRuntimeSupport` but throws an error (with
`error.code === "ASUPERSYNC_BROWSER_UNSUPPORTED_RUNTIME"`) when unsupported.
`createBrowserRuntime` calls this automatically.

#### `createUnsupportedRuntimeError(diagnostics): Error`

Creates a typed error from a failing diagnostics snapshot. The returned error
has `.code` and `.diagnostics` properties.

---

## `@asupersync/browser-core` — Low-Level Bindings

Use this package when you need raw ABI handles, the WASM metadata JSON, or
direct control over JSON-serialised communication with the WASM boundary.

### Initialization

#### `init(input?): Promise<unknown>`

Asynchronously loads the WASM module. Must be awaited before calling any
ABI functions. Calling `init` multiple times is safe — subsequent calls
return the same promise and do not re-fetch the WASM binary. ABI functions
called before `init` resolves will throw with a `TypeError` from the
wasm-bindgen runtime.

`input` accepts the same types as `WebAssembly.instantiateStreaming`: a
`URL`, `RequestInfo`, `Response`, `BufferSource`, or a
`WebAssembly.Module`. Defaults to `asupersync_bg.wasm` relative to the
package.

```ts
import init from "@asupersync/browser-core";
await init();
```

#### `initSync(module): InitOutput`

Synchronous variant for pre-compiled modules. Prefer `init` in most cases.

### ABI Functions

All ABI functions are synchronous after `init()` has been awaited. They take
typed JavaScript values and return `Outcome<T>` envelopes — the JSON
serialisation to and from WASM happens automatically inside the wrapper.

Both snake_case names and camelCase aliases are exported:

| snake_case | camelCase |
|-----------|-----------|
| `runtime_create` | `runtimeCreate` |
| `runtime_close` | `runtimeClose` |
| `scope_enter` | `scopeEnter` |
| `scope_close` | `scopeClose` |
| `task_spawn` | `taskSpawn` |
| `task_join` | `taskJoin` |
| `task_cancel` | `taskCancel` |
| `fetch_request` | `fetchRequest` |
| `websocket_open` | `websocketOpen` |
| `websocket_send` | `websocketSend` |
| `websocket_recv` | `websocketRecv` |
| `websocket_close` | `websocketClose` |
| `websocket_cancel` | `websocketCancel` |
| `abi_version` | `abiVersion` |
| `abi_fingerprint` | `abiFingerprint` |

#### `runtime_create(consumerVersion?): Outcome<RuntimeHandle>`

Creates a new root runtime. All subsequent operations require a handle
obtained from this call (or from handles derived from it).

```ts
import { runtimeCreate } from "@asupersync/browser-core";
const outcome = runtimeCreate();
```

#### `runtime_close(handle, consumerVersion?): Outcome<void>`

Closes a runtime and all its regions/tasks. All child regions must have been
closed or quiesced first.

#### `scope_enter(request, consumerVersion?): Outcome<RegionHandle>`

Opens a new region under a runtime or an existing region:

```ts
interface ScopeEnterRequest {
  parent: RuntimeHandle | RegionHandle | HandleRef;
  label?: string;
}
```

```ts
import { scopeEnter } from "@asupersync/browser-core";
const region = scopeEnter({ parent: runtimeHandle, label: "my-scope" });
```

#### `scope_close(handle, consumerVersion?): Outcome<void>`

Closes a region. Blocks until all tasks inside have quiesced.

#### `task_spawn(request, consumerVersion?): Outcome<TaskHandle>`

Registers a new tracked task inside a region:

```ts
interface TaskSpawnRequest {
  scope:        RegionHandle | HandleRef;
  label?:       string;
  cancel_kind?: string;  // default cancellation kind for this task
}
```

#### `task_join(handle, outcome, consumerVersion?): Outcome<WasmValue>`

Joins a task with its result. The `outcome` argument is the result of the
asynchronous work performed outside the runtime:

```ts
import { taskJoin } from "@asupersync/browser-core";
const joinResult = taskJoin(taskHandle, { outcome: "ok", value: "done" });
```

#### `task_cancel(request, consumerVersion?): Outcome<void>`

Requests cancellation of a task:

```ts
interface TaskCancelRequest {
  task:     TaskHandle | HandleRef;
  kind:     string;
  message?: string;
}
```

#### `abi_version(): AbiVersion`

Returns the current ABI major/minor version from the loaded WASM module.

#### `abi_fingerprint(): number`

Returns a 64-bit hash of the ABI symbol table, used to detect signature drift.

### Handle Classes

`browser-core` exports the raw handle classes that all higher-level packages
build on:

```ts
class BaseHandle {
  readonly kind:       HandleKind;
  readonly slot:       number;
  readonly generation: number;
  toJSON(): HandleRef;
}

class RuntimeHandle extends BaseHandle {
  close(consumerVersion?: AbiVersion | null): Outcome<void>;
  enterScope(label?: string, consumerVersion?: AbiVersion | null): Outcome<RegionHandle>;
}

class RegionHandle extends BaseHandle {
  close(consumerVersion?: AbiVersion | null): Outcome<void>;
  enterScope(label?: string, consumerVersion?: AbiVersion | null): Outcome<RegionHandle>;
  spawnTask(options?: Omit<TaskSpawnRequest, "scope">, consumerVersion?: AbiVersion | null): Outcome<TaskHandle>;
  fetchRequest(options: Omit<FetchRequest, "scope">, consumerVersion?: AbiVersion | null): Outcome<FetchHandle>;
  openWebSocket(url: string, protocols?: string[], consumerVersion?: AbiVersion | null): Outcome<TaskHandle>;
}

class TaskHandle extends BaseHandle {
  join(outcome: Outcome, consumerVersion?: AbiVersion | null): Outcome<WasmValue>;
  cancel(kind: string, message?: string, consumerVersion?: AbiVersion | null): Outcome<void>;
}

class CancellationToken extends BaseHandle {} // wraps a WASM-side cancel_token handle
class FetchHandle      extends BaseHandle {} // wraps a WASM-side fetch_request handle
```

### Constants and Factories

```ts
BUDGET_BOUNDS: {
  pollQuota:    { min: 1,     max: 1_000_000 };
  deadlineMs:   { min: 0,     max: 86_400_000 };
  priority:     { min: 0,     max: 255 };
  cleanupQuota: { min: 0,     max: 1_000_000 };
}

CANCELLATION_PHASE_ORDER: readonly ["requested", "draining", "finalizing", "completed"]

ERROR_CODES: readonly ["capability_denied", "invalid_handle", "decode_failure",
                       "compatibility_rejected", "internal_failure"]

RECOVERABILITY_LEVELS: readonly ["transient", "permanent", "unknown"]

// Outcome factory (also exported from @asupersync/browser):
Outcome.ok(value)
Outcome.err(code, recoverability, message)
Outcome.cancelled(cancellation)
Outcome.panicked(message)
```

### Raw WASM Bindings

`rawBindings` gives access to the underlying JSON-string ABI for debugging,
testing, and custom wrapper layers:

```ts
const rawBindings: Readonly<{
  init: typeof init;
  runtime_create(consumerVersionJson?: string): string;
  runtime_close(handleJson: string, consumerVersionJson?: string): string;
  scope_enter(requestJson: string, consumerVersionJson?: string): string;
  scope_close(handleJson: string, consumerVersionJson?: string): string;
  task_spawn(requestJson: string, consumerVersionJson?: string): string;
  task_join(handleJson: string, outcomeJson: string, consumerVersionJson?: string): string;
  task_cancel(requestJson: string, consumerVersionJson?: string): string;
  fetch_request(requestJson: string, consumerVersionJson?: string): string;
  websocket_open(requestJson: string, consumerVersionJson?: string): string;
  websocket_send(requestJson: string, consumerVersionJson?: string): string;
  websocket_recv(requestJson: string, consumerVersionJson?: string): string;
  websocket_close(requestJson: string, consumerVersionJson?: string): string;
  websocket_cancel(requestJson: string, consumerVersionJson?: string): string;
  abi_version(): string;
  abi_fingerprint(): number;
}>;
```

---

## TypeScript Types Reference

All types below are exported from both `@asupersync/browser` and
`@asupersync/browser-core`.

```ts
// WASM module initialisation input
type InitInput =
  | RequestInfo | URL | Response | BufferSource
  | WebAssembly.Module
  | Promise<RequestInfo | URL | Response | BufferSource | WebAssembly.Module>;

// ABI versioning
interface AbiVersion { major: number; minor: number; }

// Execution budget
interface Budget {
  pollQuota:    number;
  deadlineMs:   number;
  priority:     number;
  cleanupQuota: number;
}

// Cancellation lifecycle phases (in order)
type CancellationPhase = "requested" | "draining" | "finalizing" | "completed";

// Error classification
type ErrorCode =
  | "capability_denied"    // effect was blocked by capability policy
  | "invalid_handle"       // handle was stale, wrong kind, or not found
  | "decode_failure"       // JSON payload failed to deserialise
  | "compatibility_rejected" // ABI version mismatch
  | "internal_failure";    // catch-all for unexpected runtime errors

type Recoverability = "transient" | "permanent" | "unknown";

// Handle kinds
type HandleKind = "runtime" | "region" | "task" | "cancel_token" | "fetch_request";

// Opaque handle reference (serialisable)
interface HandleRef { kind: HandleKind; slot: number; generation: number; }

// ABI error shape
interface AbiFailure {
  code:            ErrorCode;
  recoverability:  Recoverability;
  message:         string;
}

// Cancellation details
interface AbiCancellation {
  kind:           string;
  phase:          CancellationPhase | string;
  origin_region:  string;
  origin_task:    string | null;
  timestamp_nanos: number;
  message:        string | null;
  truncated:      boolean;
}

// Union of all types that can cross the WASM boundary as values
type WasmValue = undefined | boolean | number | string | Uint8Array | HandleLike;

// Four-valued result type
type Outcome<T = WasmValue, E = AbiFailure> =
  | { outcome: "ok";        value: T }
  | { outcome: "err";       failure: E }
  | { outcome: "cancelled"; cancellation: AbiCancellation }
  | { outcome: "panicked";  message: string };

// Request shapes for scope/task/fetch/websocket operations
interface ScopeEnterRequest   { parent: RuntimeHandle | RegionHandle | HandleRef; label?: string; }
interface TaskSpawnRequest    { scope: RegionHandle | HandleRef; label?: string; cancel_kind?: string; }
interface TaskCancelRequest   { task: TaskHandle | HandleRef; kind: string; message?: string; }
interface FetchRequest        { scope: RegionHandle | HandleRef; url: string; method: string; body?: Uint8Array | ArrayBuffer | ArrayBufferView | number[]; }
interface WebSocketOpenRequest   { scope: RegionHandle | HandleRef; url: string; protocols?: string[]; }
interface WebSocketSendRequest   { socket: TaskHandle | HandleRef; value: WasmValue; }
interface WebSocketRecvRequest   { socket: TaskHandle | HandleRef; }
interface WebSocketCloseRequest  { socket: TaskHandle | HandleRef; reason?: string; }
interface WebSocketCancelRequest { socket: TaskHandle | HandleRef; kind: string; message?: string; }
```

---

## Fetch API

HTTP requests are issued as tracked operations inside a region. The fetch is
associated with the region's lifetime — if the region closes, the outstanding
request is cancelled via `AbortController`.

```ts
// Via @asupersync/browser RegionHandle:
const result = scope.fetchRequest({
  url: "https://api.example.com/data",
  method: "POST",
  body: new TextEncoder().encode(JSON.stringify({ key: "value" })),
});

if (result.outcome !== "ok") {
  // handle error, cancellation, or panic
  return;
}

const fetchHandle = result.value; // FetchHandle — the request is in flight
```

The `body` field accepts `Uint8Array`, `ArrayBuffer`, `ArrayBufferView`, or a
`number[]` of byte values.

### Low-level via `browser-core`

```ts
import { fetchRequest } from "@asupersync/browser-core";

const outcome = fetchRequest({
  scope:  regionHandle,
  url:    "https://api.example.com/data",
  method: "GET",
});
```

---

## WebSocket API

WebSocket connections are opened as tasks inside a region. Each socket gets a
`TaskHandle` from `openWebSocket` / `websocket_open`.

### Lifecycle

```ts
// 1. Open a WebSocket connection
const socketOutcome = scope.openWebSocket(
  "wss://stream.example.com/events",
  ["v1.protocol"],
);
if (socketOutcome.outcome !== "ok") { /* handle */ }
const socketTask = socketOutcome.value; // TaskHandle

// 2. Send a message
import { websocketSend } from "@asupersync/browser-core";
const sendResult = websocketSend({
  socket: socketTask,
  value:  "hello",
});

// 3. Receive a message
import { websocketRecv } from "@asupersync/browser-core";
const recvResult = websocketRecv({ socket: socketTask });
if (recvResult.outcome === "ok") {
  console.log("received:", recvResult.value);
}

// 4. Close gracefully
import { websocketClose } from "@asupersync/browser-core";
const closeResult = websocketClose({ socket: socketTask, reason: "done" });

// 5. Cancel (force-close with a cancellation kind)
import { websocketCancel } from "@asupersync/browser-core";
const cancelResult = websocketCancel({ socket: socketTask, kind: "user" });
```

`websocket_send` / `websocket_recv` accept `WasmValue` payloads, which means
you can send `string`, `number`, `boolean`, `Uint8Array`, or nested handle
references.

---

## React Integration (`@asupersync/react`)

`@asupersync/react` provides a React Context provider and hooks that are safe
under React StrictMode (which double-invokes effects in development).

### Quick Setup

Wrap your application root with `ReactRuntimeProvider`:

```tsx
import { ReactRuntimeProvider } from "@asupersync/react";

export default function App() {
  return (
    <ReactRuntimeProvider>
      <YourApp />
    </ReactRuntimeProvider>
  );
}
```

Pass `runtimeOptions` to customise WASM loading:

```tsx
<ReactRuntimeProvider runtimeOptions={{ wasmInput: "/wasm/asupersync_bg.wasm" }}>
  {children}
</ReactRuntimeProvider>
```

### Hooks

#### `useReactRuntime(): BrowserRuntime`

Returns the current `BrowserRuntime`. Throws (for React Suspense / Error
Boundary capture) if the runtime is not ready yet.

```tsx
import { useReactRuntime } from "@asupersync/react";

function MyComponent() {
  const runtime = useReactRuntime();
  // runtime is always non-null here
}
```

#### `useReactRuntimeContext(): ReactRuntimeContextValue`

Returns the full context, including status, error, and a `reload` function:

```ts
interface ReactRuntimeContextValue {
  status:      ReactRuntimeStatus; // "idle" | "initializing" | "ready" | "failed"
  diagnostics: ReactRuntimeSupportDiagnostics;
  runtime:     BrowserRuntime | null;
  error:       Error | null;
  reload():    void;
}
```

```tsx
import { useReactRuntimeContext } from "@asupersync/react";

function StatusBanner() {
  const { status, error, reload } = useReactRuntimeContext();
  if (status === "failed") {
    return <button onClick={reload}>Retry</button>;
  }
  return <span>{status}</span>;
}
```

#### `useReactRuntimeDiagnostics(): ReactRuntimeSupportDiagnostics`

Returns the diagnostics object (browser capability checks, support reason).

#### `useReactScope(options?): ReactScopeState`

Opens a region scope tied to the component lifecycle. The scope is
automatically closed on unmount.

```ts
interface ReactScopeOptions {
  label?:           string;
  consumerVersion?: AbiVersion | null;
}

interface ReactScopeState {
  status: "idle" | "opening" | "ready" | "failed";
  scope:  RegionHandle | null;
  error:  Error | null;
  close(): void;
}
```

```tsx
import { useReactScope } from "@asupersync/react";

function WorkComponent() {
  const { status, scope, error } = useReactScope({ label: "work" });
  if (status !== "ready" || !scope) {
    return null;
  }
  // scope is a RegionHandle — spawn tasks, issue fetch, etc.
  return <button onClick={() => scope.fetchRequest({ url: "/api", method: "GET" })}>
    Fetch
  </button>;
}
```

### Cancellation in React

Use `CancellationToken` to cancel inflight work when a component unmounts or
the user navigates away:

```tsx
import { useEffect } from "react";
import { useReactScope, CancellationToken } from "@asupersync/react";

function SearchBox({ query }: { query: string }) {
  const { scope } = useReactScope({ label: "search" });

  useEffect(() => {
    if (!scope) return;
    const task = scope.spawnTask({ label: "search-task" });
    if (task.outcome !== "ok") return;

    // When the effect re-runs (query changes) or component unmounts:
    return () => {
      CancellationToken.user("query changed").cancel(task.value);
    };
  }, [scope, query]);

  return <input value={query} />;
}
```

### Support Detection in React

```ts
import { detectReactRuntimeSupport } from "@asupersync/react";

const diag = detectReactRuntimeSupport();
// diag.packageName === "@asupersync/react"
```

---

## Next.js Integration (`@asupersync/next`)

`@asupersync/next` re-exports everything from `@asupersync/browser` and adds
adapters for the three Next.js rendering boundaries.

### Boundary Model

| Boundary | Direct runtime | Adapter |
|----------|---------------|---------|
| Client component (hydrated) | ✅ supported | `createNextBootstrapAdapter` |
| Server component / route handler | ❌ not supported | `createNextServerBridgeAdapter` |
| Edge runtime | ❌ not supported | `createNextEdgeBridgeAdapter` |

**Never call `createBrowserRuntime` from server components, route handlers, or
`middleware.ts`.** Gate all direct runtime calls behind the
`"use client"` directive.

### Bootstrap Adapter (Client Boundary)

The `NextClientBootstrapAdapter` manages the full hydration lifecycle:
`server_rendered → hydrating → hydrated → runtime_ready`.

Create one adapter per page/route and drive it through the lifecycle:

```ts
import {
  createNextBootstrapAdapter,
  type NextClientBootstrapOptions,
} from "@asupersync/next";

const adapter = createNextBootstrapAdapter({
  initialRouteSegment: "/dashboard",
  label: "dashboard-runtime",
  popstatePreservesRuntime: true,
  onLogEvent: (event, snapshot) => {
    console.log("[asupersync]", event.action, snapshot.phase);
  },
});

// Typical lifecycle driven from a client component:
adapter.beginHydration();     // SSR has started: "server_rendered" → "hydrating"
adapter.completeHydration();  // React hydration done: "hydrating" → "hydrated"
const scopeOutcome = await adapter.initializeRuntime(); // loads WASM, creates runtime+scope
                                                         // → "runtime_ready"

// For the common case (all three in one call):
const scopeOutcome2 = await adapter.hydrateAndInitialize();
```

The full public API of `NextClientBootstrapAdapter`:

```ts
class NextClientBootstrapAdapter {
  // Lifecycle transitions (each returns a log event)
  beginHydration(): NextBootstrapLogEvent;           // → "hydrating"
  completeHydration(): NextBootstrapLogEvent;        // → "hydrated"
  runtimeInitFailed(reason: string): NextBootstrapLogEvent;    // → "runtime_failed"
  cancelBootstrap(reason: string): NextBootstrapLogEvent;      // → "runtime_failed"
  hydrationMismatch(reason: string): NextBootstrapLogEvent;    // → "runtime_failed"
  recover(action: NextBootstrapRecoveryAction): NextBootstrapLogEvent;
  navigate(type: NextNavigationType, routeSegment: string): NextBootstrapLogEvent;
  hotReload(): NextBootstrapLogEvent;
  cacheRevalidated(): NextBootstrapLogEvent;
  close(reason?: string): NextBootstrapLogEvent;

  // Async runtime initialisation
  initializeRuntime(): Promise<Outcome<RegionHandle>>;  // → "runtime_ready"
  ensureRuntimeReady(): Promise<Outcome<RegionHandle>>; // idempotent: advances all phases
  hydrateAndInitialize(): Promise<Outcome<RegionHandle>>; // alias for ensureRuntimeReady()

  // Accessors
  snapshot(): NextBootstrapSnapshot;
  events(): readonly NextBootstrapLogEvent[];
  currentRuntime(): BrowserRuntime | null;
  currentScope(): RegionHandle | null;
  createLogFields(event: NextBootstrapLogEvent, overrides?: NextBootstrapLogFieldOverrides): Record<string, string>;
}
```

Navigation handling:

```ts
// Soft navigation (preserves runtime):
adapter.navigate("soft_navigation", "/dashboard/settings");

// Hard navigation (tears down and re-initializes runtime):
adapter.navigate("hard_navigation", "/other-page");

// Popstate (browser back/forward):
adapter.navigate("popstate", "/previous");

// Hot-reload (development only):
adapter.hotReload();
```

Recovery from failed initialisation:

```ts
adapter.recover("retry_runtime_init");     // re-attempt from "hydrated"
adapter.recover("reset_to_hydrating");     // roll back to "hydrating"
```

### Server Bridge Adapter

Use `NextServerBridgeAdapter` (or the standalone functions) in server
components and API route handlers:

```ts
import { createNextServerBridgeAdapter } from "@asupersync/next";

// In a server component or API route:
export async function GET() {
  const bridge = createNextServerBridgeAdapter({
    renderEnvironment: "node_server",
    routeSegment: "/api/items",
  });

  // bridge.diagnostics().directRuntimeSupported === false

  // Build a request envelope to send to the client:
  const req = bridge.createRequest("fetchItems", { filter: "active" });

  // Build a response from an outcome:
  const response = bridge.ok({ items: [] });
  // or:
  const errResponse = bridge.err("database unavailable");

  return Response.json(response);
}
```

`NextServerBridgeAdapter` API:

```ts
class NextServerBridgeAdapter {
  diagnostics(): NextServerBridgeDiagnostics;
  createLogFields(overrides?: NextBootstrapLogFieldOverrides): Record<string, string>;
  createRequest<T>(operation: string, payload: T, options?: NextServerBridgeRequestOptions): NextServerBridgeRequest<T>;
  ok<T>(payload: T): NextServerBridgeResponse<T>;
  err(message: string): NextServerBridgeResponse;
  cancelled(message?: string): NextServerBridgeResponse;
  fromOutcome<T>(outcome: Outcome<T, AbiFailure>): NextServerBridgeResponse<T>;
  unwrapResponse<T>(response: NextServerBridgeResponse<T>): T; // throws on non-ok
  unsupportedRuntimeError(): NextServerBridgeRuntimeError;
}
```

### Edge Bridge Adapter

```ts
import { createNextEdgeBridgeAdapter } from "@asupersync/next";

// In middleware.ts or an edge route:
const bridge = createNextEdgeBridgeAdapter({
  renderEnvironment: "edge_runtime",
});

// bridge.diagnostics().directRuntimeSupported === false
const response = bridge.ok({ ok: true });
```

`NextEdgeBridgeAdapter` mirrors the `NextServerBridgeAdapter` API with
`NextEdgeBridgeDiagnostics` and `NextEdgeBridgeResponse` types.

### Utility Functions

```ts
// Determine boundary mode for a given render environment
nextBoundaryModeForEnvironment(environment: NextRenderEnvironment): NextBoundaryMode;

// Determine fallback strategy
nextRuntimeFallbackForEnvironment(environment: NextRenderEnvironment): NextRuntimeFallback;

// Human-readable explanation of the fallback
nextRuntimeFallbackReason(environment: NextRenderEnvironment): string;

// Detect support for a specific Next.js target ("client" | "server" | "edge")
detectNextRuntimeSupport(target?: NextRuntimeTarget): NextRuntimeSupportDiagnostics;

// Throws if target is not supported
assertNextRuntimeSupport(target?: NextRuntimeTarget): NextRuntimeSupportDiagnostics;

// Standalone wrappers for response creation (functional style alternative to adapter classes):
createNextServerBridgeResponseFromOutcome<T>(outcome, options?): NextServerBridgeResponse<T>;
createNextEdgeBridgeResponseFromOutcome<T>(outcome, options?): NextEdgeBridgeResponse<T>;

// Unwrap response payloads (throw on non-ok):
unwrapNextServerBridgeResponse<T>(response: NextServerBridgeResponse<T>): T;
unwrapNextEdgeBridgeResponse<T>(response: NextEdgeBridgeResponse<T>): T;
```

### Next.js Bootstrap Snapshot

`adapter.snapshot()` returns a detailed telemetry object:

```ts
const s = adapter.snapshot();
// s.phase                  — current bootstrap phase
// s.runtimeInitAttempts    — how many init attempts have been made
// s.runtimeInitSuccesses   — how many succeeded
// s.hardNavigationCount    — hard page navigations since last init
// s.phaseHistory           — full ordered list of phase transitions
// s.lastError              — last error string if any
// s.activeScopeGeneration  — current scope generation counter
```

---

## Error Handling

### Error codes

All ABI errors set `failure.code` to one of:

| Code | Meaning | Recoverability |
|------|---------|---------------|
| `capability_denied` | Effect blocked by capability policy | permanent |
| `invalid_handle` | Handle is stale, wrong kind, or unknown | permanent |
| `decode_failure` | JSON payload failed to parse or validate | permanent |
| `compatibility_rejected` | ABI version mismatch between consumer and producer | permanent |
| `internal_failure` | Unexpected internal runtime error | unknown |

### Checking outcomes

```ts
const outcome = runtimeCreate();
if (outcome.outcome === "err") {
  const { code, recoverability, message } = outcome.failure;
  if (recoverability === "transient") {
    // safe to retry
  }
}
```

### Unsupported runtime errors

When `createBrowserRuntime` is called outside a supported browser environment
it throws an error with `.code === "ASUPERSYNC_BROWSER_UNSUPPORTED_RUNTIME"`.
The error also carries a `.diagnostics` property with the full
`BrowserRuntimeSupportDiagnostics` snapshot.

```ts
import {
  createBrowserRuntime,
  BROWSER_UNSUPPORTED_RUNTIME_CODE,
} from "@asupersync/browser";

try {
  const runtime = await createBrowserRuntime();
} catch (err) {
  if (err.code === BROWSER_UNSUPPORTED_RUNTIME_CODE) {
    console.warn(err.diagnostics.guidance);
  }
}
```

---

## ABI Versioning

The WASM ABI uses a `major.minor` version scheme. A `consumerVersion` can be
passed to every ABI call to lock the caller to a specific version:

```ts
import { runtimeCreate } from "@asupersync/browser-core";
const outcome = runtimeCreate({ major: 1, minor: 0 });
```

Compatibility rules:

| Situation | Result |
|-----------|--------|
| Same major + same minor | `exact` — fully compatible |
| Same major + consumer minor > producer minor | `backward_compatible` |
| Same major + consumer minor < producer minor | `incompatible` |
| Different major | `major_mismatch` — incompatible |

Minor bumps are additive only (new fields, new symbols, behavioural
relaxations). Major bumps indicate breaking changes.

Introspect the loaded ABI at runtime:

```ts
import { abiVersion, abiFingerprint } from "@asupersync/browser-core";

const version = abiVersion();
// { major: 1, minor: 0 }

const fingerprint = abiFingerprint();
// 4558451663113424898  (u64 hash of the symbol table)
```

The `abi-metadata.json` artifact is available at:

```ts
import abiMetadata from "@asupersync/browser-core/abi-metadata.json";
// { abi_version: { major: 1, minor: 0 }, abi_signature_fingerprint_v1: 4558451663113424898, profile: "prod" }
```

---

## See Also

- [`docs/wasm_abi_contract.md`](wasm_abi_contract.md) — ABI versioning rules and break taxonomy
- [`docs/wasm_typescript_type_model_contract.md`](wasm_typescript_type_model_contract.md) — TypeScript type system contracts
- [`docs/wasm_cancellation_abortsignal_contract.md`](wasm_cancellation_abortsignal_contract.md) — Cancellation semantics
- [`docs/wasm_browser_scheduler_semantics.md`](wasm_browser_scheduler_semantics.md) — Scheduler and cooperative scheduling
- [`docs/wasm_react_reference_patterns.md`](wasm_react_reference_patterns.md) — React canonical patterns
- [`docs/wasm_nextjs_template_cookbook.md`](wasm_nextjs_template_cookbook.md) — Next.js App Router templates
- [`docs/wasm_troubleshooting_compendium.md`](wasm_troubleshooting_compendium.md) — Diagnostics and troubleshooting
- [`docs/wasm_dx_error_taxonomy.md`](wasm_dx_error_taxonomy.md) — Error code reference
