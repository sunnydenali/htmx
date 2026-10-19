# htmx — System Architecture Document

**Repository:** `sunnydenali/htmx` · **Branch:** `uat` · **Commit:** `fa978b24` · **Version:** 2.0.11

This document is derived from the repository source on the `uat` branch, principally `src/htmx.js`, plus build scripts, tests, and CI configuration. Diagrams use Mermaid and render on GitHub.

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Repository Layout & Build Topology](#2-repository-layout--build-topology)
3. [High-Level System Context](#3-high-level-system-context)
4. [Core Module Decomposition](#4-core-module-decomposition)
5. [Public API & Internal API](#5-public-api--internal-api)
6. [Configuration Model](#6-configuration-model)
7. [Bootstrap & Initialization Workflow](#7-bootstrap--initialization-workflow)
8. [Node Processing Pipeline](#8-node-processing-pipeline)
9. [Trigger System](#9-trigger-system)
10. [Attribute Inheritance & Extended Selectors](#10-attribute-inheritance--extended-selectors)
11. [Request Lifecycle](#11-request-lifecycle)
12. [Request Synchronization](#12-request-synchronization)
13. [Input Collection & Parameter Encoding](#13-input-collection--parameter-encoding)
14. [Response Handling](#14-response-handling)
15. [Swap Engine](#15-swap-engine)
16. [Out-of-Band, Partial & Preserved Swaps](#16-out-of-band-partial--preserved-swaps)
17. [Settle Phase & Element Lifecycle](#17-settle-phase--element-lifecycle)
18. [Event System](#18-event-system)
19. [History Subsystem](#19-history-subsystem)
20. [Extension Architecture](#20-extension-architecture)
21. [Internal Data Model](#21-internal-data-model)
22. [Security Controls](#22-security-controls)
23. [Build & Distribution](#23-build--distribution)
24. [Testing Architecture](#24-testing-architecture)
25. [CI/CD Workflow](#25-cicd-workflow)
26. [Consumer Integration](#26-consumer-integration)
27. [Design Decisions & Trade-offs](#27-design-decisions--trade-offs)
28. [Glossary](#28-glossary)

---

## 1. Executive Summary

htmx is a dependency-free browser library that lets HTML elements issue HTTP requests in response to DOM events and replace selected DOM regions with server-returned HTML. Its design embraces hypermedia: the server returns HTML fragments and can control client behaviour through `HX-*` response headers.

At runtime, htmx is a single IIFE in `src/htmx.js`, exposing a global `htmx` API. It scans `hx-*` and `data-hx-*` attributes, binds triggers, serializes values, sends `XMLHttpRequest`s, handles response headers/statuses, swaps DOM fragments, and processes inserted content. Custom events and an extension interface provide the principal customization points. The runtime itself has no third-party runtime dependency.

[↑ Back to Table of Contents](#table-of-contents)

## 2. Repository Layout & Build Topology

```text
htmx/
  src/
    htmx.js                 Core browser runtime (single IIFE; JSDoc-typed)
    htmx.esm.d.ts           ESM TypeScript declarations
    htmx.test.ts            Type-level smoke test
  dist/                     Committed distribution artifacts
    ext/                    Legacy 1.x extensions retained for old URLs
  scripts/
    dist.sh                 Creates IIFE, minified, gzip, AMD, CJS, ESM outputs
    generate-web-types.mjs  Generates JetBrains HTML/CSS/JS metadata from site docs
    www.sh                  Copies runtime/tests/dependencies into website assets
    sha.sh                  SRI hash utility
    update-sha.sh           Updates the README integrity value
  editors/jetbrains/        Generated IDE web-types metadata
  test/
    attributes/             Attribute-focused browser tests
    core/                   Runtime feature tests
    util/                   Test helpers
    manual/                 Browser/manual regression scenarios
    index.html              Manual Mocha test page
  www/                      Zola website source and static test assets
  .github/workflows/ci.yml  Node-based test workflow
  package.json              Package metadata, scripts, development dependencies
  web-test-runner.config.mjs Browser automation configuration
```

```mermaid
flowchart LR
  S[src/htmx.js] --> DS[scripts/dist.sh]
  DS --> IIFE[dist/htmx.js]
  DS --> MIN[dist/htmx.min.js and gzip]
  DS --> AMD[dist/htmx.amd.js]
  DS --> CJS[dist/htmx.cjs.js]
  DS --> ESM[dist/htmx.esm.js]
  ESM --> TSC[TypeScript declaration generation]
  TSC --> TYPES[dist/htmx.esm.d.ts]
  DOCS[www/content docs] --> WEB[generate-web-types.mjs]
  WEB --> IDE[editors/jetbrains/htmx.web-types.json]
  S -. npm run www .-> SITE[www/static and website theme assets]
```

The core code is intentionally kept in one source file. The website is a separate Zola static site; tests exercise the runtime in real browser engines.

[↑ Back to Table of Contents](#table-of-contents)

## 3. High-Level System Context

```mermaid
flowchart LR
  USER[User] -->|click, submit, change, scroll| DOM[HTML document with hx-* attributes]
  DOM <--> HTMX[htmx browser runtime]
  HTMX -->|XMLHttpRequest + HX-* request headers| SERVER[Hypermedia application server]
  SERVER -->|HTML fragment + HX-* response headers| HTMX
  HTMX -->|DOM updates, browser events| DOM
  HTMX <--> WEBAPI[Browser APIs: DOM, XHR, History, sessionStorage]
  EXT[Optional separately packaged extensions] <-->|extension hooks| HTMX
```

| Boundary | Contract |
|---|---|
| Page → htmx | `hx-*` / `data-hx-*` attributes, `<meta name="htmx-config">`, public `htmx.*` API |
| htmx → server | XHR; request metadata such as `HX-Request`, `HX-Trigger`, `HX-Target`, `HX-Current-URL`, and `HX-Boosted` |
| Server → htmx | HTML response body, status code, and optional `HX-*` response headers |
| Extensions ↔ htmx | `htmx.defineExtension()` with lifecycle, encoding, response-transform, and swap hooks |

[↑ Back to Table of Contents](#table-of-contents)

## 4. Core Module Decomposition

Though implemented in one closure, `src/htmx.js` is sectioned into logical subsystems:

```mermaid
flowchart TB
  API[Public htmx API]
  INIT[Initialization and configuration]
  NODE[Node processing and lifecycle]
  TRIG[Trigger parsing and dispatch]
  SELECT[Attribute resolution and extended selectors]
  AJAX[Request construction and XHR]
  INPUT[Input collection, validation, encoding]
  RESPONSE[Response handling and headers]
  SWAP[HTML parsing and swap engine]
  SETTLE[Settle tasks, reprocessing, focus and scroll]
  EVENTS[Custom event and logging system]
  HISTORY[History and sessionStorage cache]
  EXT[Extension registry and hooks]

  API --> NODE & AJAX & SWAP & EVENTS & EXT
  INIT --> NODE & HISTORY
  NODE --> TRIG & SELECT & EVENTS
  TRIG --> AJAX
  AJAX --> INPUT & SELECT & EVENTS & EXT
  AJAX --> RESPONSE
  RESPONSE --> SWAP & HISTORY & EVENTS & EXT
  SWAP --> SETTLE & EVENTS & EXT
  SETTLE --> NODE
  EVENTS --> EXT
  HISTORY --> SWAP
```

A key feedback loop is **swap → settle → processNode**: newly inserted fragments are processed just like initial markup, enabling declarative behaviour to compose across server responses.

[↑ Back to Table of Contents](#table-of-contents)

## 5. Public API & Internal API

The public `htmx` object exposes event, request, DOM, swap, extension, debugging, configuration, and utility APIs. Functions are assigned to the object after declaration. Extensions receive a narrower `internalAPI` object through their `init` hook.

```mermaid
classDiagram
  class htmx {
    +version
    +config
    +onLoad(callback)
    +process(elt)
    +on(target,event,listener)
    +off(target,event,listener)
    +trigger(elt,event,detail)
    +ajax(verb,path,context)
    +values(elt,verb)
    +find(root,selector)
    +findAll(root,selector)
    +closest(elt,selector)
    +swap(target,content,spec,options)
    +defineExtension(name,extension)
    +removeExtension(name)
    +addClass/removeClass/toggleClass/takeClass
    +logAll/logNone
  }
  class internalAPI {
    <<extension init API>>
    addTriggerHandler
    getInputValues
    getHeaders
    getTarget
    getSwapSpecification
    makeFragment
    querySelectorExt
    triggerEvent
    withExtensions
    swap
  }
  class HtmxExtension {
    <<interface>>
    init(api)
    getSelectors()
    onEvent(name,event)
    transformResponse(text,xhr,elt)
    isInlineSwap(style)
    handleSwap(style,target,fragment,settleInfo)
    encodeParameters(xhr,parameters,elt)
  }
  htmx ..> internalAPI : supplies to extension.init
  internalAPI ..> HtmxExtension : extension contract
```

[↑ Back to Table of Contents](#table-of-contents)

## 6. Configuration Model

Runtime defaults are defined in `htmx.config`. During initialization, `mergeMetaConfig()` merges JSON from `<meta name="htmx-config" content="...">` over defaults. Applications may also modify `htmx.config` at runtime.

Important defaults include:

| Area | Defaults |
|---|---|
| History | enabled; cache size 10; cache miss fetches server unless configured to refresh |
| Swap | `innerHTML`; 0 ms swap delay; 20 ms settle delay |
| Request methods | `get` and `delete` parameters go in URL by default |
| Response status | 204: no swap; 2xx/3xx: swap; 4xx/5xx: no swap and error |
| Security | same-origin requests only; `allowEval` and `allowScriptTags` enabled by default |
| UI classes | `htmx-request`, `htmx-indicator`, `htmx-added`, `htmx-swapping`, `htmx-settling` |
| Transitions | global View Transitions disabled by default |
| Credentials | `withCredentials` false; timeout 0 |

```mermaid
flowchart LR
  DEFAULTS[Built-in htmx.config defaults] --> MERGE[mergeMetaConfig at ready time]
  META[htmx-config meta JSON] --> MERGE
  MERGE --> LIVE[Live runtime configuration]
  JS[Application JavaScript] --> LIVE
```

[↑ Back to Table of Contents](#table-of-contents)

## 7. Bootstrap & Initialization Workflow

```mermaid
sequenceDiagram
  participant Script as htmx IIFE
  participant Doc as document
  participant Init as ready callback
  participant Body as document.body
  participant Win as window
  Script->>Script: create public API, config, internal API
  Script->>Doc: register DOMContentLoaded readiness flag
  Script->>Init: ready(init)
  Init->>Init: mergeMetaConfig()
  Init->>Doc: insert indicator CSS if enabled
  Init->>Body: processNode(body)
  Init->>Body: register htmx:abort handler
  Init->>Win: wrap onpopstate for htmx history entries
  Init->>Win: setTimeout(0), fire htmx:load
```

`ready()` accounts for both normal page loading and scripts inserted after the document is already loaded. The initial `processNode(body)` scans the document and installs behaviour.

[↑ Back to Table of Contents](#table-of-contents)

## 8. Node Processing Pipeline

`processNode(elt)` resolves an element or selector, rejects disabled subtrees, scans for actionable elements, detects changed attributes, installs `hx-on` handlers, and initializes nodes.

```mermaid
flowchart TD
  A([processNode]) --> B[Resolve element]
  B --> C{Disabled by hx-disable?}
  C -- yes --> D[cleanUpElement] --> Z([return])
  C -- no --> E[Hash root attributes]
  E --> F[Find actionable descendants and extension selectors]
  F --> G[Check each candidate for disablement and changed hash]
  G --> H[Deinitialize changed nodes and collect for init]
  H --> I[Find and install hx-on handlers]
  I --> J[initNode for changed candidates]
  J --> Z
```

`attributeHash()` provides idempotency: unchanged elements are not rebound on repeated processing. If attributes change, `deInitNode()` removes old listeners/timers before reinitialization. `initNode()` chooses explicit HTTP verbs, boosted links/forms, trigger-only behaviour, and submit-button tracking.

[↑ Back to Table of Contents](#table-of-contents)

## 9. Trigger System

`hx-trigger` is parsed by a tokenizer and parser into `HtmxTriggerSpecification` objects. Supported behaviour includes event filters, `once`, `changed`, `consume`, delay/throttle, `from`, `target`, queue policy, polling (`every`), `revealed`, and `intersect`.

```mermaid
flowchart TD
  A([initNode]) --> B[getTriggerSpecs]
  B --> C{explicit hx-trigger?}
  C -- no --> D[Choose element default: form submit, input change, otherwise click]
  C -- yes --> E[Tokenize and parse clauses/modifiers]
  D & E --> F[addTriggerHandler]
  F --> G{trigger kind}
  G -->|load| H[Run once after optional delay]
  G -->|every interval| I[Recursive timer; stop when detached/cancelled]
  G -->|revealed| J[Scroll/resize visibility check]
  G -->|intersect| K[IntersectionObserver]
  G -->|DOM event| L[Install event listener]
  H & I & J & K & L --> M[Filter / once / changed / delay / throttle / queue]
  M --> N[Request handler]
```

| Element kind | Default trigger |
|---|---|
| `<form>` | `submit` |
| Submit/button inputs | `click` |
| `input`, `textarea`, `select` | `change` |
| Other elements | `click` |

[↑ Back to Table of Contents](#table-of-contents)

## 10. Attribute Inheritance & Extended Selectors

Most `hx-*` values are resolved from the triggering element up its ancestor chain. `hx-inherit`, `hx-disinherit`, and `config.disableInheritance` control inheritance. The traversal supports shadow-root parent relationships.

```mermaid
flowchart TD
  A([Resolve attribute]) --> B[Walk from element toward ancestors]
  B --> C{Attribute found?}
  C -- no --> B
  C -- yes --> D{Inheritance disabled?}
  D -- yes --> E{Explicit hx-inherit permits it?}
  E -- yes --> V[Use value]
  E -- no --> N[Absent]
  D -- no --> F{hx-disinherit blocks it?}
  F -- yes --> N
  F -- no --> V
```

The extended selector engine supports standard CSS plus relative forms such as `this`, `closest`, `find`, `next`, `previous`, `document`, `window`, `body`, `root`, `host`, and `global`. It is used by target, include, indicator, event-source, and other selector-valued attributes.

[↑ Back to Table of Contents](#table-of-contents)

## 11. Request Lifecycle

`issueAjaxRequest()` is the central request coordinator. It validates the trigger/target, applies submitter overrides, supports confirmation and prompts, enforces synchronization, gathers input, emits mutable/cancellable events, verifies the URL, configures XHR, sends the request, and dispatches response/error events.

```mermaid
sequenceDiagram
  autonumber
  participant User
  participant Trigger as Trigger handler
  participant Core as issueAjaxRequest
  participant Events as htmx event system
  participant XHR as XMLHttpRequest
  participant Server
  participant Response as handleAjaxResponse
  participant Swap as swap()
  User->>Trigger: DOM event
  Trigger->>Core: verb, path, element, event
  Core->>Core: resolve target and submitter overrides
  Core->>Events: htmx:confirm / prompt hooks
  Core->>Core: hx-sync, collect values, validate, filter
  Core->>Events: htmx:configRequest (mutable request config)
  Core->>Core: verify URL and configure XHR/headers
  Core->>Events: htmx:beforeRequest, then htmx:beforeSend
  Core->>XHR: send encoded parameters
  XHR->>Server: HTTP request with HX-* metadata
  Server-->>XHR: HTML body, status, HX-* response headers
  XHR-->>Core: onload
  Core->>Response: response handler
  Response->>Swap: response-selected DOM update
  Core->>Events: afterRequest / afterOnLoad
```

```mermaid
flowchart TD
  A([Request]) --> B{Trigger still in document?}
  B -- no --> Z([resolve without sending])
  B -- yes --> C{Target valid?}
  C -- no --> X1[htmx:targetError]
  C -- yes --> D[Apply submitter verb/path]
  D --> E[Confirmation event and optional prompt/confirm]
  E --> F[Apply hx-sync strategy]
  F --> G[Collect values and validate]
  G --> H[htmx:configRequest]
  H --> I{Cancelled or validation errors?}
  I -- yes --> X2[Halt request and release lock]
  I -- no --> J[Build URL/body and verify same-origin policy]
  J --> K{URL allowed?}
  K -- no --> X3[htmx:invalidPath]
  K -- yes --> L[Configure XHR and fire beforeRequest]
  L --> M{beforeRequest cancelled?}
  M -- yes --> X2
  M -- no --> N[Indicators and disabled controls]
  N --> O[beforeSend and xhr.send]
```

[↑ Back to Table of Contents](#table-of-contents)

## 12. Request Synchronization

Per-element internal state tracks `xhr`, whether it is abortable, and queued callbacks. `hx-sync` coordinates requests against a selected element.

```mermaid
stateDiagram-v2
  [*] --> Idle
  Idle --> InFlight: create XHR and store on element data
  InFlight --> InFlight: drop strategy discards incoming request
  InFlight --> Queued: queue first / last / all
  InFlight --> Aborting: replace strategy or abortable request
  Aborting --> InFlight: start replacement request
  Queued --> InFlight: endRequestLock dequeues next request
  InFlight --> Idle: completion, error, timeout, or abort
```

| Strategy | Effect while request is active |
|---|---|
| `drop` | Discard the new request (default) |
| `abort` | Abortable state governs whether current work can be replaced; otherwise drop |
| `replace` | Trigger `htmx:abort`, then continue with the new request |
| `queue first` | Retain the first pending request |
| `queue last` | Keep only the most recent pending request |
| `queue all` | Retain all pending requests |

[↑ Back to Table of Contents](#table-of-contents)

## 13. Input Collection & Parameter Encoding

`getInputValues()` returns validation errors, a native `FormData`, and a proxy view used for compatibility. Non-GET requests include the related form; `hx-include`, submitter values, `hx-vals`, `hx-vars`, and `hx-params` modify the effective parameter set.

```mermaid
flowchart TD
  A([getInputValues]) --> B{Verb is non-GET?}
  B -- yes --> C[Collect related form into priority values]
  B -- no --> D
  C --> D[Collect triggering element]
  D --> E[Include last clicked submitter when applicable]
  E --> F[Collect hx-include targets and descendant controls]
  F --> G[Priority form/submitter values override]
  G --> H[Return FormData, proxy, validation errors]
  H --> I[Merge ajax context values and hx-vars/hx-vals]
  I --> J[Apply hx-params filter]
  J --> K{Method uses URL parameters?}
  K -- yes --> L[URL-encode into query]
  K -- no --> M{Extension encoder provided value?}
  M -- yes --> N[Use extension body]
  M -- no --> O{multipart encoding?}
  O -- yes --> P[Send FormData]
  O -- no --> Q[URL-encoded request body]
```

Native browser validation is performed when applicable. Validation events are emitted, and errors halt transmission with `htmx:validation:halted`.

[↑ Back to Table of Contents](#table-of-contents)

## 14. Response Handling

`handleAjaxResponse()` first gives listeners a chance to cancel `htmx:beforeOnLoad`, dispatches server-triggered events, handles redirect/refresh headers, determines history changes, resolves status-code behaviour, applies retarget/reswap/reselect directives, and then optionally swaps.

```mermaid
flowchart TD
  A([XHR response]) --> B[htmx:beforeOnLoad]
  B --> C[Dispatch HX-Trigger]
  C --> D{HX-Location?}
  D -- yes --> D1[Issue htmx GET navigation]
  D -- no --> E{HX-Redirect or HX-Refresh?}
  E -- yes --> E1[Browser navigation/reload]
  E -- no --> F[Resolve history update]
  F --> G[Resolve responseHandling by HTTP status]
  G --> H[Apply HX-Retarget, HX-Reswap, HX-Reselect]
  H --> I[htmx:beforeSwap; listeners may mutate details]
  I --> J{shouldSwap?}
  J -- no --> K[Emit response error if configured]
  J -- yes --> L[Extension transformResponse]
  L --> M[Save current history if needed]
  M --> N[swap(target, response, specification)]
  N --> O[After-swap and after-settle trigger headers]
```

Response status handling is configurable with regular-expression-like status match strings. Response headers provide server-side controls such as `HX-Trigger`, `HX-Location`, `HX-Redirect`, `HX-Refresh`, `HX-Retarget`, `HX-Reswap`, `HX-Reselect`, `HX-Push-Url`, and `HX-Replace-Url`.

[↑ Back to Table of Contents](#table-of-contents)

## 15. Swap Engine

`swap()` parses HTML into a detached fragment, optionally runs a View Transition, performs OOB and partial swaps before the main swap, applies the selected swap style, restores focus/selection, updates the title, and settles new nodes.

```mermaid
flowchart TD
  A([swap target, content, spec]) --> B{View Transition enabled and supported?}
  B -- yes --> C[Wrap swap in startViewTransition]
  B -- no --> D
  C --> D{swap delay configured?}
  D -- yes --> E[Wait swap delay]
  D -- no --> F
  E --> F[Capture focus and selection]
  F --> G{Text content style?}
  G -- yes --> H[Set target.textContent]
  G -- no --> I[Parse response to DocumentFragment]
  I --> J[Select OOB swaps]
  J --> K[Process hx-swap-oob elements]
  K --> L[Process partial templates]
  L --> M[Apply hx-select]
  M --> N[Handle hx-preserve elements]
  N --> O[Main swapWithStyle]
  H & O --> P[Restore focus/selection and title]
  P --> Q[Fire htmx:afterSwap; schedule settle]
  Q --> R[Run settle tasks, htmx:afterSettle, scrolling]
```

Built-in styles include `innerHTML`, `outerHTML`, `beforebegin`, `afterbegin`, `beforeend`, `afterend`, `delete`, and `none`. `textContent` is handled directly by `swap()`. Extensions may implement custom swap styles through `handleSwap()`.

[↑ Back to Table of Contents](#table-of-contents)

## 16. Out-of-Band, Partial & Preserved Swaps

```mermaid
flowchart TD
  RESP[Parsed response fragment] --> SELECTOOB[hx-select-oob selection]
  SELECTOOB --> OOB[Find hx-swap-oob elements]
  OOB --> PARTIAL[Find template[hx] partials]
  PARTIAL --> SELECT[hx-select main-fragment selection]
  SELECT --> PRESERVE[Match hx-preserve elements by id]
  PRESERVE --> MAIN[Perform main target swap]
  MAIN --> RESTORE[Restore preserved nodes]
```

- **Out-of-band swap:** `hx-swap-oob` directs a response node to update a different target, normally selected by its id; missing targets emit an error event.
- **Partial templates:** `<hx-…>` response tags are rewritten to `<template hx type="…">` before parsing. `type="partial"` templates can target a node using `hx-target` or their id and trigger partial lifecycle events. Other types emit `htmx:processTemplate` for integrations.
- **Preserved elements:** `hx-preserve` nodes with matching ids are moved through the swap rather than recreated. Where supported, `moveBefore` is used to preserve browser-managed state; otherwise the existing node replaces the incoming placeholder.

The processing order is OOB selection/swaps, partials, main-fragment selection, preservation, and the main swap.

[↑ Back to Table of Contents](#table-of-contents)

## 17. Settle Phase & Element Lifecycle

The settle phase delays final attribute application to support CSS transitions, processes inserted nodes, restores focus, and emits lifecycle events.

```mermaid
sequenceDiagram
  participant Swap as swap()
  participant DOM
  participant Settle as settle tasks
  participant Process as processNode
  Swap->>DOM: Insert fragment nodes and add htmx-added
  Swap->>DOM: Temporarily retain matching id attributes for transition
  Swap->>Settle: Queue attribute restoration and load tasks
  Note over Swap,Settle: default settle delay is 20 ms
  Settle->>DOM: Apply final attributes; remove htmx-settling
  Settle->>Process: Process inserted element subtree
  Process-->>DOM: Install htmx behaviours on new content
  Settle->>DOM: Focus autofocus element; emit htmx:load/afterSettle
  Settle->>DOM: Apply scroll/show options
```

```mermaid
stateDiagram-v2
  [*] --> Unprocessed
  Unprocessed --> Initialized: processNode / initNode
  Initialized --> Initialized: attributes unchanged
  Initialized --> Reinitialized: attribute hash changed
  Initialized --> Requesting: trigger starts request
  Requesting --> Swapping: response selected for swap
  Swapping --> Settling: nodes inserted
  Settling --> Initialized: tasks complete and new nodes processed
  Initialized --> CleanedUp: removed/replaced node cleanup
  CleanedUp --> [*]
```

[↑ Back to Table of Contents](#table-of-contents)

## 18. Event System

`triggerEvent()` creates bubbling, cancellable, composed `CustomEvent`s. The return value reflects cancellation and extension vetoes. Most events also dispatch a kebab-case alias. Events provide application hooks, cancellation points, extension integration, and error reporting.

```mermaid
flowchart TD
  A([triggerEvent]) --> B[Attach detail.elt and create CustomEvent]
  B --> C[Optional logger]
  C --> D{Error detail?}
  D -- yes --> E[Log and emit htmx:error]
  D -- no --> F
  E --> F[Dispatch canonical event]
  F --> G[Dispatch kebab-case alias when applicable]
  G --> H[Invoke active extensions' onEvent hooks]
  H --> I[Return not-cancelled result]
```

Major event groups include node processing (`beforeProcessNode`, `afterProcessNode`, `load`), request (`trigger`, `confirm`, `configRequest`, `beforeRequest`, `beforeSend`, `afterRequest`), response/swap (`beforeOnLoad`, `beforeSwap`, `afterSwap`, `afterSettle`), validation, errors, history, and OOB/partial swaps. `hx-on:*` and `hx-on-*` attributes attach inline handlers, subject to `allowEval`.

[↑ Back to Table of Contents](#table-of-contents)

## 19. History Subsystem

History uses the browser History API plus a bounded `sessionStorage` cache. The saved item contains normalized URL, selected HTML content, title, and scroll position. `hx-history-elt` selects the snapshot region; absent that attribute, the body is used. `hx-history="false"` opts out of snapshotting the page content.

```mermaid
sequenceDiagram
  participant Response
  participant History as htmx history subsystem
  participant Browser as History API
  participant Store as sessionStorage
  Response->>History: Determine push/replace update
  History->>Store: Save current HTML/title/scroll unless hx-history=false
  History->>Browser: pushState or replaceState
  Browser-->>History: popstate for htmx entry
  History->>Store: Find cached snapshot
  alt cache hit
    History->>History: Swap cached content and restore title/scroll
  else cache miss
    History->>Response: GET path with HX-History-Restore-Request
    Response-->>History: Full page or history fragment
    History->>History: Swap history element and emit restore events
  end
```

If storage is unavailable or quota is exceeded, history gracefully falls back; on quota errors the cache is reduced and retried. A configured cache miss may trigger a full reload instead of an XHR restore.

[↑ Back to Table of Contents](#table-of-contents)

## 20. Extension Architecture

Extensions are registered in a runtime map with `defineExtension(name, implementation)`. The implementation is merged with defaults. `hx-ext` activates extensions on an element and inherited ancestors; `ignore:name` excludes one. Extensions live in separate packages; `dist/ext/` contains legacy htmx 1.x files retained to preserve older URLs and is not actively maintained for v2.

```mermaid
classDiagram
  class ExtensionRegistry {
    -extensions: Map
    +defineExtension(name, partial)
    +removeExtension(name)
    +getExtensions(element)
    +withExtensions(element, callback)
  }
  class HtmxExtension {
    <<interface>>
    init(api)
    getSelectors()
    onEvent(name,event)
    transformResponse(text,xhr,elt)
    isInlineSwap(style)
    handleSwap(style,target,fragment,settleInfo)
    encodeParameters(xhr,parameters,elt)
  }
  ExtensionRegistry --> HtmxExtension
```

```mermaid
flowchart LR
  SCAN[Node scan] -->|getSelectors| EXT[Extension]
  EVENT[Custom event] -->|onEvent| EXT
  PARAMS[Request body] -->|encodeParameters| EXT
  RESPONSE[Response text] -->|transformResponse| EXT
  SWAP[Custom swap style] -->|handleSwap / isInlineSwap| EXT
  TEMPLATE[Unknown template type] -->|htmx:processTemplate| EXT
```

[↑ Back to Table of Contents](#table-of-contents)

## 21. Internal Data Model

Per-element state is stored on the element under `htmx-internal-data`. Request and swap flow objects are plain objects described with JSDoc typedefs in the source.

```mermaid
classDiagram
  class HtmxNodeInternalData {
    initHash
    boosted
    listenerInfos
    timeout
    triggeredOnce
    delayed
    throttle
    lastValue
    loaded
    path
    verb
    polling
    lastButtonClicked
    requestCount
    xhr
    queuedRequests
    abortable
    firstInitCompleted
  }
  class HtmxRequestConfig {
    boosted
    useUrlParams
    formData
    parameters
    headers
    elt
    target
    verb
    errors
    withCredentials
    timeout
    path
    triggeringEvent
  }
  class HtmxResponseInfo {
    xhr
    target
    requestConfig
    pathInfo
    successful
    failed
    keepIndicators
  }
  class HtmxSwapSpecification {
    swapStyle
    swapDelay
    settleDelay
    transition
    ignoreTitle
    scroll
    show
    focusScroll
  }
  class HtmxSettleInfo {
    tasks
    elts
    title
  }
  HtmxResponseInfo --> HtmxRequestConfig
  HtmxResponseInfo --> HtmxSwapSpecification
```

[↑ Back to Table of Contents](#table-of-contents)

## 22. Security Controls

```mermaid
flowchart TD
  EVAL[Expression evaluation: filters, hx-vars, hx-on] --> ALLOW{config.allowEval?}
  ALLOW -- no --> ERR[htmx:evalDisallowedError; use fallback]
  ALLOW -- yes --> RUN[Evaluate configured expression]
  SCRIPT[Scripts in response fragment] --> SCRIPTS{config.allowScriptTags?}
  SCRIPTS -- no --> STRIP[Remove script tags]
  SCRIPTS -- yes --> NORMALIZE[Normalize executable scripts; optionally apply nonce]
  URL[Final request URL] --> ORIGIN{selfRequestsOnly and same origin?}
  ORIGIN -- no --> BLOCK[htmx:invalidPath]
  ORIGIN -- yes --> VALIDATE[htmx:validateUrl event]
  HISTORY[Page snapshot] --> PRIVATE{hx-history=false present?}
  PRIVATE -- yes --> SKIP[Do not cache snapshot]
  PRIVATE -- no --> CACHE[Cache in sessionStorage]
```

Other relevant protections include a same-origin request default, configurable script/style CSP nonces, selector escaping for generated id selectors, and the history-cache opt-out for sensitive pages. Since `allowEval` and script interpretation are enabled by default in this version, applications with strict CSP requirements should configure them appropriately and review server-returned HTML policy.

[↑ Back to Table of Contents](#table-of-contents)

## 23. Build & Distribution

`npm run dist` invokes the shell distribution builder, generates declarations with TypeScript, and generates JetBrains web-types from website documentation.

```mermaid
flowchart TD
  RUN[npm run dist] --> SH[scripts/dist.sh]
  SH --> COPY[Copy src/htmx.js to dist/htmx.js]
  COPY --> MIN[Uglify minification and gzip]
  MIN --> AMD[AMD wrapper]
  AMD --> CJS[CommonJS wrapper]
  CJS --> ESM[ES module wrapper]
  RUN --> TYPES[tsc declaration generation]
  ESM --> TYPES
  TYPES --> DT[dist/htmx.esm.d.ts]
  RUN --> WT[generate-web-types.mjs]
  DOC[www/content attribute/reference/event docs] --> WT
  WT --> IDE[editors/jetbrains/htmx.web-types.json]
```

| Package field | Target |
|---|---|
| `main` | `dist/htmx.esm.js` |
| `types` | `dist/htmx.esm.d.ts` |
| `jsdelivr`, `unpkg` | `dist/htmx.min.js` |
| `web-types` | `editors/jetbrains/htmx.web-types.json` |
| License | 0BSD |

The source is also copied into website static assets by `npm run www`; the website itself is served locally with Zola as documented in `www/README.md`.

[↑ Back to Table of Contents](#table-of-contents)

## 24. Testing Architecture

The repository has both a manual Mocha browser page and automated browser tests. `web-test-runner.config.mjs` loads the runtime and test helpers into a browser page, runs attribute/core tests with Playwright, and enables coverage for `src/htmx.js`.

```mermaid
flowchart LR
  MANUAL[test/index.html: Mocha BDD browser suite] --> HARNESS[Test harness]
  AUTO[Web Test Runner + Playwright] --> HARNESS
  HARNESS --> CHAI[Chai / chai-dom]
  HARNESS --> SINON[Sinon XHR mocking]
  HARNESS --> SOCKET[mock-socket]
  HARNESS --> UTIL[test/util/util.js]
  HARNESS --> CORE[test/core/**/*.js]
  HARNESS --> ATTR[test/attributes/**/*.js]
  AUTO --> COV[Coverage: src/htmx.js]
  TYPE[TypeScript checkJs and declaration generation] --> CI[Validation]
```

The manual test page also links manual and scratch scenarios. Automated suites cover core behaviour and individual attributes; browser-specific scripts support Chromium, Firefox, and WebKit.

[↑ Back to Table of Contents](#table-of-contents)

## 25. CI/CD Workflow

`.github/workflows/ci.yml` defines a Node CI job on Ubuntu 22.04, using Node 20, `npm ci`, and `npm test` for configured push and pull-request branches. `npm test` runs lint, type checking, and the Chromium browser suite. The package also provides all-browser test commands.

```mermaid
flowchart LR
  EVENT[Push or pull request on configured branch] --> CHECKOUT[actions/checkout]
  CHECKOUT --> NODE[Set up Node.js 20]
  NODE --> INSTALL[npm ci]
  INSTALL --> TEST[npm test]
  TEST --> LINT[ESLint]
  LINT --> TYPES[TypeScript checkJs]
  TYPES --> BROWSER[Playwright Chromium tests]
```

The workflow's branch filters are `master`, `dev`, `htmx-2.0`, and `v2.0v2.0`; `uat` is not listed as a push target in the workflow configuration.

[↑ Back to Table of Contents](#table-of-contents)

## 26. Consumer Integration

```mermaid
flowchart TB
  subgraph Distribution
    CDN[CDN script: dist/htmx.min.js]
    NPM[npm package: ESM / CJS / AMD]
  end
  subgraph BrowserPage[Application page]
    CONFIG[Optional htmx-config meta]
    MARKUP[HTML annotated with hx-*]
    RUNTIME[htmx runtime]
  end
  subgraph Server[Application server]
    ROUTE[Route/action]
    HTML[HTML fragment response]
    HEADERS[HX-* response directives]
  end
  CDN --> RUNTIME
  NPM --> RUNTIME
  CONFIG --> RUNTIME
  MARKUP --> RUNTIME
  RUNTIME -->|XHR + HX request metadata| ROUTE
  ROUTE --> HTML
  ROUTE --> HEADERS
  HTML --> RUNTIME
  HEADERS --> RUNTIME
  RUNTIME --> MARKUP
```

The backend is intentionally unconstrained: it may be implemented in any language/framework, provided it handles the HTTP request and returns appropriate HTML. It can use `HX-Request` to distinguish htmx requests and respond with a fragment where appropriate.

[↑ Back to Table of Contents](#table-of-contents)

## 27. Design Decisions & Trade-offs

| Decision | Benefit | Trade-off |
|---|---|---|
| Single IIFE source file | Small, dependency-free, broadly embeddable distribution | Logical modules are not enforced by file boundaries |
| Declarative attributes | Behaviour is visible in HTML and integrates with server-rendered views | Attribute inheritance and selector resolution add complexity |
| Custom events as hooks | One mechanism supports app callbacks, cancellation, extensions, and observability | Large event surface; event ordering is part of compatibility |
| XMLHttpRequest transport | Upload/download progress, abort, familiar browser support | No native fetch streaming pipeline |
| Swap plus settle phases | Supports progressive CSS transitions and post-insertion processing | Requires temporary classes and deferred tasks |
| sessionStorage history snapshots | Fast back/forward restoration without a server round trip on cache hit | Storage quota and sensitive-content concerns; opt-out exists |
| FormData-backed parameter proxy | Supports multiple values and files with compatibility object semantics | Proxy behaviour is more complex than a plain object |
| Externalized extensions | Keeps core small and lets integrations evolve independently | Extension availability/versioning is managed separately |

[↑ Back to Table of Contents](#table-of-contents)

## 28. Glossary

| Term | Meaning |
|---|---|
| **Boost** | Progressive enhancement of eligible links/forms into AJAX navigation via `hx-boost`. |
| **OOB** | Out-of-band response fragment swap targeting a different DOM node via `hx-swap-oob`. |
| **Partial** | Response template of type `partial`, independently targeting a node. |
| **Settle** | Post-insertion phase for attribute reconciliation, node processing, focus, and lifecycle events. |
| **Trigger specification** | Parsed event/polling rule built from one `hx-trigger` clause. |
| **Internal data** | Per-element runtime state stored under `htmx-internal-data`. |
| **Extended selector** | htmx selector dialect layered over CSS selectors. |
| **286 response** | A status recognized by htmx to cancel polling for the triggering element. |
| **DUMMY_ELT** | Sentinel target used to surface a target resolution error rather than accidentally swapping into the body. |

[↑ Back to Table of Contents](#table-of-contents)
