


# BluffCharge Methodology

This document describes the design, techniques, and rationale behind the BluffCharge system — a browser-based sincerity appraisal engine for distinguishing genuine users from insincere, automated, or falsified clients.

### 📜 Self-Appraisal and Audit Workflow

The core methodology involves client-side self appraisal augmented by server-seeded integrity anchors. This two-part mechanism enables both lightweight in-browser evaluation and strong post-hoc verification of sincerity.

#### 1. Self-Appraisal on the Client

- Each client evaluates a series of passive and active browser behavior tests.
- The output is reduced to a sincerity score or appraisal vector.
- This appraisal is transmitted alongside an encrypted or hashed log of input signals.

#### 2. Anchoring with Server Heartbeat

- Periodic heartbeat signals from the server inject a session-specific seed.
- This seed is required for generating valid hashes for the self appraisal submission.
- The server retains a timestamped record of each heartbeat for audit traceability.

#### 3. Retrospective Reappraisal

- At any point, the server can re-evaluate the visitor’s sincerity score using the original input log and server-held heartbeat.
- A mismatch between self-appraised and server-computed results indicates deception or log tampering.
- This deferred audit mechanism allows for scalable verification without disrupting user experience in real time.

This architecture balances low-latency interaction on the frontend with strong anti-tampering guarantees for high-trust systems.

## Overview

BluffCharge leverages passive browser fingerprinting, dynamic chore execution, client self-appraisal, and server-audited hash validation to detect subtle discrepancies in runtime environments. It is designed for use with logged-in users, delivering high-confidence identity and behavior analysis without imposing CAPTCHA friction.

---

## Key Components

### ✅ Self-Appraisal with Hash Auditing

- Client executes "chores" (dispatched JS tests) and generates a local appraisal.
- Results are hashed with a server-provided seed (from heartbeat log) to prevent falsified logs.
- Appraisal reports and hash pairs are submitted for server-side or ONNX-based validation.
- Audits can retroactively request re-verification using immutable seeds.

### 🧠 ONNX Model Integration

- A lightweight ONNX model receives a feature vector derived from client-side tests.
- Model determines plausibility and sincerity of behavior, with thresholds for confidence.
- Server can perform deeper analysis if suspicion is flagged.

### 🧬 Feature Vector Schema

The ONNX model receives a normalized feature vector for each session that includes values from the various chore categories. Each value is consistently preprocessed on the client, hashed (for audit alignment), and interpreted on the server.

Example feature vector structure:

| Feature Name                     | Type     | Description                                                               | Normalization |
|----------------------------------|----------|---------------------------------------------------------------------------|----------------|
| `js_engine.optional_chaining`    | boolean  | Support for optional chaining operator (`?.`)                             | 0/1 |
| `font_rendering.clear_type_score`| float    | Entropy or pixel deviation score from subpixel rendering                  | 0–1 scaled |
| `scroll.inertia_shape`           | float    | Curve similarity to native inertia profiles                               | Z-score |
| `timezone_offset`                | integer  | Minutes offset from UTC                                                   | Centered |
| `intl_locale_match`              | boolean  | True if reported locale matches IP-expected region                        | 0/1 |
| `mouse.entropy`                  | float    | Entropy in recent mouse move segment                                      | 0–1 |
| `storage.quota_estimate_ratio`   | float    | Ratio of reported vs expected quota for device/browser                    | Clipped 0–1 |
| `canvas.text_baseline_drift`     | float    | Baseline jitter under repeated renders                                    | Mean deviation |
| `mutation_observer_delay`        | float    | Average delay between mutation and observer callback                      | ms scaled |
| `clipboard.type_count`           | integer  | Number of MIME types in clipboardData.types during passive paste          | Integer range |

The feature vector is submitted as part of the appraisal payload. If an audit is later triggered, the same vector is recomputed from logs for comparison. Missing features default to neutral baseline values during inference.

### 🔄 Dynamic Task Dispatch

- Tasks (or "chores") are described in `chore_registry.json`.
- Each chore collects one or more metrics — from rendering, environment, timing, etc.
- Chores are designed to be passive and silently executed in the background.

### 🎯 Challenge-Based Chore Tasks

In addition to passive measurement, BluffCharge supports the dynamic dispatch of invisible "challenge chores" — tasks designed to elicit low-effort client response patterns without disrupting the user experience. These challenges are not visible UI prompts but lightweight scripts whose completion timing, fidelity, or entropy may help differentiate sincere human browsing from automation.

Examples include:

- **Scroll Completion Echo**: A `requestAnimationFrame` callback waits for natural scroll end behavior after a chore-initiated `scrollTo()` trigger. Non-native clients often complete immediately or with non-inertial curves.
- **Pointer Curve Response**: A challenge chore might request the client to "move the cursor smoothly left and back" via an animation loop and monitor entropy. A bot’s gesture will often lack human-style granularity or hesitancy.
- **Focus Trap Handling**: Dispatch a short-lived hidden input focus trap and observe how the environment escapes or responds to DOM focus shifts.
- **Idle Recheck Trigger**: Wait a few seconds after chore batch dispatch, and reissue the exact same test. Highly synthetic clients often return the exact same output with zero jitter or memory effect.

These tasks form a spectrum between passive fingerprinting and full interaction, offering meaningful behavioral samples while staying below the threshold of user awareness or annoyance.

---

## Passive Detection Categories

### 🧪 JavaScript Engine Quirks

### 📜 JavaScript Syntax Feature Tests

These tests validate the claimed JavaScript engine version by evaluating syntax features introduced in specific ECMAScript editions. Discrepancies between claimed user agent and actual support reveal spoofed or outdated engines.

#### Syntax Features and Engine Mapping

- **Nullish Coalescing (`??`)**: Introduced in ES2020. Failure to parse indicates older JS engine.
- **Optional Chaining (`?.`)**: ES2020 syntax; unsupported in many headless environments and older emulations.
- **Private Class Fields (`#field`)**: Introduced in ES2022; strong indicator of modern engine.
- **Top-Level Await**: Only supported in module contexts under modern browsers.
- **Static Class Blocks**: Introduced in ES2022; rare in polyfills.
- **Array Grouping (`|>` and `||=` proposals)**: Feature gate indicators for bleeding-edge engines.
- **Async Generator Functions**: Syntax and behavior differences in older implementations.
- **Try-Catch Binding Behavior**: Scope leakage or strict handling reveals older engines.

Each feature is tested via `eval()` inside a try-catch block. Parse or runtime errors are logged as part of the self-assessment vector.

These tests are lightweight, silent, and cannot be polyfilled—making them robust indicators of JavaScript engine authenticity.

- **Optional Chaining Support**: Detects presence of `obj?.prop` syntax (ES2020+).
- **Nullish Coalescing Operator**: Checks `a ?? b` support, often used in modern conditional logic.
- **BigInt Literal Parsing**: Validates support for `123n` numeric literals.
- **Symbol.asyncIterator Presence**: Reveals support for async iterator constructs.
- **import.meta Handling**: Determines if `import.meta` is accessible in module scope.
- **String.prototype.matchAll**: Useful indicator of ES2020-level string handling.
- **Structured Clone Support**: Edge cases like cloning with embedded functions reveal engine quirks.
- **Function.prototype.toString Drift**: Native vs polyfilled function formatting inconsistency.
- **Reflect and Proxy Fidelity**: Proxy traps and reflect behavior diverge subtly under headless environments.
- **Error Message Text and Type**: `try { eval(...) }` or undefined operations yield version-specific error strings.
- **Async Microtask Timing**: Order of `Promise.resolve().then(...)` relative to other tasks can differ subtly.
- **Try/Catch Scope Quirks**: Older engines leak catch-scoped variables.
- **Eval Scoping Behavior**: Evaluate isolated block scope pollution or unexpected globals.
- **Global Object Identification**: Compare `globalThis`, `window`, `self`, and `global` resolution.

### 🖼️ Rendering and Font Fidelity

- Canvas rendering and pixel pattern hashing
- Subpixel layout based on OS and display stack
- Font fallback behavior under uncommon font names
- WebGL shader timing or result drift
- AudioContext fingerprinting (entropy in frequency bins)

#### 🪟 Subpixel Rendering and Display Fidelity

These tests focus on ultra-fine rendering and layout behaviors that vary across OS, browser, compositor stack, and monitor type.

- **Font Subpixel Anti-Aliasing Differences**: Render small gray text and inspect for ClearType, CoreText, or grayscale-only anti-alias patterns.
- **Canvas Text Metrics**: Measure `ctx.measureText()` against actual pixel bounding box and font fallback metrics.
- **Text Baseline Jitter**: Draw text at subpixel-aligned vertical positions and detect jitter or aliasing in baseline render.
- **Font Rendering Order and Fallback**: Deliberately render using rare fonts and detect font fallback sequence via canvas hashing.
- **Device Pixel Ratio vs True DPI**: Cross-check `window.devicePixelRatio` with physical screen resolution and scroll behavior for inconsistencies.
- **Gradient Banding or Dithering Test**: Render subtle grayscale or hue gradient and detect visible banding to infer display depth.
- **Gamma Curve Behavior**: Measure brightness linearity via offscreen canvas rendering and pixel inspection.
- **Native Scroll Inertia Curve**: Measure scroll momentum behavior and compare to platform-native curves.
- **Rounded Corner Anti-Aliasing Pattern**: Use `border-radius` and blur detection to fingerprint compositor behavior and rendering pipeline.

### 🕹️ Behavior and Movement Fingerprinting

- Pointer entropy and drift
- Scroll deceleration realism
- Selection behavior
- Visibility/tab switching patterns
- Input caret blink rate

### ⏱️ Timing and System Clock Checks

- Clock skew vs server timestamp
- Timezone offset inconsistencies
- `Intl.DateTimeFormat` locale mismatch
- JSON.parse benchmark timing
- Layout reflow jitter

### 📦 Storage and Permission Behavior

- localStorage/sessionStorage persistence across reloads
- Quota estimate accuracy
- IndexedDB error surface
- navigator.plugins and mimeTypes coherence

### 🌐 Network & Prefetch Behavior

- Artificial 204 latency tests
- WebSocket echo jitter
- Beacon delivery drift
- Prefetch bait traps
- ServiceWorker install timing (if supported)

### 🧭 Platform Consistency Clues

- navigator.webdriver flag
- `navigator.hardwareConcurrency` realism
- Touch support vs claimed device type
- matchMedia for `(forced-colors)` detection
- Scrollbar width, style, and computed default layout values

### 📅 Locale & Calendar Realism

- Week start day vs locale
- Date formatting mismatch
- Right-to-left rendering quirks
- Hyphenation rules and auto-linebreak behavior

### 🧩 DOM and Isolation Fidelity

- MutationObserver vs real DOM update timing
- iframe sandbox isolation test
- `getBoundingClientRect()` alignment differences
- DOMParser edge case parsing behavior

---

## Additional Passive Fingerprinting Categories

These passive signals provide subtle but powerful insight into the sincerity and realism of the visitor’s browsing environment without requiring user interaction. All data points are eligible for inclusion in the ONNX audit model and downstream behavior validation.

### Interactivity Coherence (Passive Signals)
- **Tab/Focus Change Patterns**: Use `document.visibilityState` and `visibilitychange` events to record focus shifts over time.
- **Mouse Drift and Movement Entropy**: Monitor mouse trajectory smoothness, deviation entropy, and periodicity to detect unnatural input behavior. Sample rate, path curvature, and sudden jumps are especially informative.

### Ambient OS Integration Hints
- **Touch Event Structure**: Validate that `TouchEvent` objects include expected fields such as `radiusX`, `rotationAngle`, and `force`. Compare structure to device expectations.
- **Clipboard Metadata**: For passive paste detection, inspect `clipboardData.types` and structure for natural diversity and non-empty transfer formats.

### Accessibility & Focus Behavior
- **ARIA Live Region Fidelity**: Trigger DOM updates inside `aria-live` containers and observe reflow timing and `MutationObserver` signals.
- **Forced Colors Media Query**: Evaluate `matchMedia('(forced-colors)')` for accessibility mode hints that are platform-specific.

### Environmental Noise and Latency Signatures
- **requestIdleCallback Jitter**: Schedule multiple idle callbacks and record delay jitter distribution across call frames.
- **Canvas createImageBitmap Timing**: Decode complex image blobs and measure the latency of the resulting `createImageBitmap` instantiation.
- **DeviceMotion Noise Floor**: In supported environments, examine accelerometer noise across idle periods and correlate with expected hardware variance.
- **WebRTC ICE Candidate Entropy**: Passively generate ICE candidates using `RTCPeerConnection` and compare IP protocol types, port ranges, and NAT mapping strategies.

### Style and Layout Coherence
- **Scrollbar Dimension Detection**: Estimate scrollbar width and style using overflow-based measurement tricks to correlate platform assumptions.
- **Computed Style Default Drift**: Extract `getComputedStyle()` values from unstyled elements and detect platform-default variations in color, spacing, borders, etc.
- **Selection and Caret Defaults**: Track `selectionchange`, caret blink rate, and DOM focus traversal latency to build low-friction behavioral models.

### DOM and Rendering Lifecycle Behavior
- **MutationObserver Timing Drift**: Log the timing between DOM mutations and observer notifications, particularly on passive layout-triggering operations.
- **DOMParser Quirks**: Feed edge-case HTML into `DOMParser` and validate parse tree structures. Older engines may mishandle certain attributes or malformed tags.

### Time, Locale, and Calendar Fidelity
- **User Locale vs IP Locale**: Derive IP region on server side and compare with `Intl.DateTimeFormat().resolvedOptions().locale`.
- **System Timezone Drift**: Use `Date.getTimezoneOffset()` and compare against server guess or known offset windows.
- **Clock Desynchronization**: Measure delay and skew of multiple server roundtrips and flag inconsistencies beyond a platform-tuned threshold.
- **Week Start Convention**: Render a JS calendar component and passively check whether the first column begins on Monday, Sunday, or otherwise.

### Storage and Persistence Fidelity
- **SessionStorage vs LocalStorage Lifespan**: Confirm realistic session vs permanent data behavior under multiple reload and soft-crash tests.
- **IndexedDB Fallback Behavior**: Attempt to open IndexedDB with extreme version numbers and observe precise failure message signatures.
- **Quota Estimate Realism**: Call `navigator.storage.estimate()` and validate quota and usage bounds against expected baselines for browser/device.


## Appraisal and Integrity

- Each submission of self-assessment includes:
  - Raw metric report
  - Appraised verdict
  - Hashed commitment including prior heartbeat-seeded entropy
- Audit challenges use earlier heartbeat values, preventing tampering.
- All chore results contribute to ONNX feature vector and optional forensic storage.


## 🧪 Monitor Type and Display Hardware Inference

BluffCharge includes passive tests to infer monitor characteristics and display pipeline behavior. This class of signals is difficult to falsify because it originates from a deep interaction between OS, GPU drivers, and physical display properties.

### Display Detection Techniques

- **Subpixel Anti-Aliasing Fingerprint**: Different OS and GPU combinations apply distinct subpixel rendering (e.g., ClearType vs grayscale). We render small grayscale text to a canvas and measure RGB drift patterns.
- **Canvas Gamma and Banding Test**: Display a smooth gradient and measure canvas pixel values for nonlinear gamma or banding artifacts.
- **Device Pixel Ratio vs Scroll Ratio**: Compare `window.devicePixelRatio` with scroll delta behavior and layout rounding errors.
- **Rounded Corner Blur Profile**: Render blurred border-radius shapes and measure anti-alias falloff to infer compositor characteristics.
- **Gradient Band Count**: Count visible bands in a multi-stop CSS gradient to infer display depth (6-bit, 8-bit, etc.).
- **Text Alignment Jitter**: Repeatedly draw text at subpixel offsets and measure alignment noise.

These measurements do not require user interaction and are repeatable across sessions. They feed into the ONNX model for sincerity inference and may also serve as tie-breakers during audit reviews.

---



## 🧩 Chore Registry Format and Validation

## 🧪 Network Timing and Behavior Tests

Certain network behaviors reveal inconsistency between claimed environments and actual client stack. These passive signals are hard to spoof and require no user interaction.

### Timing-Based Heuristics

- **204 No Content Response Latency**: Send a preloaded fetch request for a static 204 resource and measure roundtrip time. Bots or proxy stacks may return results unrealistically fast.
- **WebSocket Echo Roundtrip Delay**: Establish a `WebSocket` connection and echo a token, logging response delay. Jitter, chunk boundaries, and compression flags can hint at environment.
- **Navigator.connection Inference**: Evaluate `navigator.connection.downlink`, `rtt`, and `effectiveType` against actual roundtrip performance to detect falsified values.
- **Beacon Delivery Timing**: Use `navigator.sendBeacon()` to measure background delivery lag. Bot environments sometimes short-circuit this.
- **ServiceWorker Timing Hooks**: If supported, install a `ServiceWorker` and monitor installation delay and activation pattern. Deviations from platform norms are strong fingerprinting vectors.

These timing-based features are useful for identifying virtualized, accelerated, or spoofed browsing environments.

Each client-side task — or "chore" — is described and dispatched from a central file named `chore_registry.json`. This registry defines what each chore does, how it’s measured, and what feature(s) it contributes to.

### Chore Definition Schema

Each entry in the registry follows this structure:

```json
{
  "id": "canvas_text_jitter",
  "description": "Measure baseline jitter in canvas text rendering",
  "category": "Rendering",
  "metrics": ["canvas.text_baseline_drift"],
  "execution": {
    "type": "script",
    "source": "chores/canvas_text_jitter.js"
  },
  "flags": {
    "passive": true,
    "challenge": false
  }
}
```

### Fields:

- `id`: Unique identifier for the chore.
- `description`: Human-readable explanation.
- `category`: Logical grouping (e.g. Rendering, Timing, JS Engine).
- `metrics`: List of metric names this chore produces.
- `execution`: Specifies how the test is performed.
  - `type`: `script` (JS file), `inline`, or `module`.
  - `source`: Path to or inline reference.
- `flags`:
  - `passive`: Indicates this chore requires no user interaction.
  - `challenge`: Marks if this is a challenge-style task used to probe deeper behavior.

### Validation Rules

- No duplicate chore IDs.
- All metrics must map to valid ONNX feature names.
- Flags must be explicitly set (defaults are not assumed).
- Paths must be relative and constrained to the sandbox directory.

### Registry Lifecycle

The registry is versioned alongside the ONNX model and participates in audit traceability. Older sessions can be reconstructed using the registry state active at the time of appraisal.

This design ensures chore logic is declarative, auditable, and aligned with versioned inference workflows.



### 🧪 Eval Scoping and Isolation

This section tests the isolation behavior of `eval()` within different JavaScript scope contexts. Older or falsified engines may leak or improperly share variables due to outdated scoping semantics.

#### Key Behaviors

- **Block Scope Leakage**: Wrap an `eval()` call inside a block and test whether declared variables escape their intended scope.
- **Shadowing Consistency**: Use nested `eval()` to verify correct variable shadowing behavior.
- **Strict Mode Side Effects**: Compare scoping and reassignment quirks under `"use strict"` inside and outside `eval()`.

These tests reveal both version mismatch and structural engine tampering. They are especially useful when a bot attempts to spoof modern syntax support but implements outdated semantics.

## 🧪 JS Engine Differential Behavior Tests


### 🧪 Global Object Identification

This test investigates how different environments expose or alias the global object. Discrepancies here often indicate headless clients, sandboxed runtimes, or incorrect emulation layers.

#### Key Observations

- **globalThis Consistency**: Validate that `globalThis` equals `window` in browser contexts and not `undefined`.
- **Self and Window Alias**: In a browser, `self === window` is expected. Deviation suggests non-window contexts (e.g. service workers or emulators).
- **Global Leakage**: Test if assignments to undeclared variables leak into the global object — behavior differs across strict and non-strict environments.
- **Typeof Behavior Drift**: Check that `typeof globalThis` is `'object'` and that unusual values like `typeof global` do not exist in the browser.


These patterns are easy to probe and hard to fake in layered browser stacks. They help confirm runtime boundaries and reveal hidden abstraction layers.

### 🧪 Error Message String Analysis

Subtle differences in the wording of JavaScript error messages can serve as strong indicators of engine version and environment authenticity.

#### Key Checks

- **Eval Syntax Errors**: Run deliberately malformed `eval()` strings and capture thrown error messages. Compare exact strings against known browser versions.
- **Type Errors from Undefined Methods**: Attempt to call non-existent methods and record `TypeError` messages. Browser engines differ in phrasing and formatting.
- **RangeError vs TypeError Boundaries**: Push limits on array sizes, string lengths, or recursion depth to determine which type of error is thrown and the message text.
- **Custom Error Stack Shape**: Capture multi-level thrown errors and inspect `error.stack` format, whitespace, and call site annotations.
- **Error Constructor Behavior**: Test if custom error objects with subclassed constructors are serialized consistently.

These tests are passive, unintrusive, and produce consistent results within browser families — making them ideal for verifying runtime claims and revealing impersonation via message mismatches.

These tests focus on exposing subtle behavior differences between JavaScript engines — particularly useful when a client claims to be running an evergreen browser but actually uses an outdated or falsified one. Rather than checking syntax alone, these tests examine runtime quirks and inconsistencies in edge case behavior.

### Key Tests

- **Eval Block Scope Contamination**: Evaluate whether `eval` introduces leaked variables into block scope — older engines may mishandle this.
- **Function.prototype.toString Fidelity**: Compare `toString()` results of native vs polyfilled or rewrapped functions. Some headless engines show malformed or unminified output.
- **Error Stack Format Divergence**: Capture error stack traces from a thrown exception and compare format, line numbers, and depth. These vary subtly across engine versions.
- **Proxy Trap Semantics**: Implement full `Proxy` trap coverage and test which traps are called in ambiguous situations — edge differences are detectable.
- **Microtask Queue Timing**: Queue `Promise.resolve().then(...)` and compare its resolution ordering against `setTimeout(...)`. Not all JS engines respect the same ordering spec.
- **Intl API Granularity**: Format the same date or number with slight input variations and compare results. The precision of `Intl.NumberFormat` and `Intl.DateTimeFormat` reveals underlying engine fidelity.
- **RegExp Edge Case Differences**: Test regex behavior with lookaheads, backreferences, and unicode property escapes — not all engines handle them identically.
- **Math Function Drift**: Compare outputs from `Math.acosh`, `Math.hypot`, and others using specific float inputs to detect deviation in precision.
- **Garbage Collection Timing Probe**: Schedule memory allocation patterns and infer GC pass characteristics based on latency spikes (not reliable but can reinforce suspicion).

These tests are passive, fast, and cannot be polyfilled or masked easily — making them ideal for detecting inconsistencies in falsified client stacks.

## 🧠 ONNX Training Dataset and Model Management

The ONNX model used for sincerity inference requires high-quality labeled data to perform reliably. BluffCharge defines the following standards for building and maintaining this model:

### Dataset Composition

- **Sincere Visitors**: Sessions from known legitimate users, ideally verified by account history and clean audit trails.
- **Insincere Visitors**: Captured sessions from automation tools, bots, or flagged replay/fraudulent attempts.
- **Uncertain Sessions**: Edge cases stored for human-in-the-loop review and iterative relabeling.

### Feature Vector Normalization

- Features are normalized using statistical transformations (e.g. Z-score, unit scaling) at training time.
- All runtime inputs must match this transformation pipeline exactly to ensure inference accuracy.

### Versioning and Drift Management

- Every model is versioned using a checksum and linked to its training dataset hash.
- Model versions are stored alongside audit logs for traceability.
- Retraining is triggered when feature distributions drift or detection confidence decays on labeled audit sets.

### Model Confidence Thresholds

- Confidence scores are used to triage sessions:
  - **High Confidence Genuine**: No action required.
  - **Low Confidence or Ambiguous**: Triggers audit replay or alternate challenge.
  - **High Confidence Insincere**: May flag session or trigger authentication step.

### Storage and Replay Auditability

- Every session appraisal stores its input feature vector and ONNX output.
- These are retained for potential re-evaluation after model updates.
- A shadow audit system periodically re-infers older sessions with new models to detect emerging patterns.

This training methodology ensures the ONNX layer evolves with new detection vectors, minimizes false positives, and remains explainable during forensic or legal review.

- Each submission of self-assessment includes:
  - Raw metric report
  - Appraised verdict
  - Hashed commitment including prior heartbeat-seeded entropy
- Audit challenges use earlier heartbeat values, preventing tampering.
- All chore results contribute to ONNX feature vector and optional forensic storage.

### 🔐 Replay Resistance and Audit Mechanism

A core principle of BluffCharge is its ability to detect falsified or backfilled self-appraisals through immutable audit challenge seeds.

- **Heartbeat-Sealed Log Integrity**: Each session is periodically issued a "heartbeat" — a timestamped value from the server. All subsequent appraisal logs are hashed with this seed, binding them to the specific moment of measurement.
- **Retroactive Audit Requests**: During review, the server may request revalidation of a prior log entry. The client must resubmit the same metrics, hashed using the original heartbeat value. Any inconsistency reveals falsification.
- **Time-Based Entropy Binding**: Because heartbeat values are monotonic and unique, a client cannot precompute valid hashes for future audits without knowing future seeds. This prevents spoofing of delayed or fabricated logs.
- **Audit Window Variation**: The audit may deliberately target *earlier* self-reports in a random distribution, ensuring attackers cannot predict or bias their falsification strategy.

This architecture ensures high-fidelity appraisal streams even in adversarial conditions, enabling downstream trust in ONNX inferences or human-in-the-loop forensics.

---

## Comparison to CAPTCHA

| Test Type        | User Disruption | Accuracy | Replayable | Tamper-Proof |
|------------------|------------------|----------|-------------|----------------|
| Traditional CAPTCHA | High            | Medium   | Yes         | Weak            |
| BluffCharge Chores  | None            | High     | No          | Strong          |

BluffCharge enables unobtrusive confidence scoring with retrospective auditability, dramatically improving the detection of highly evasive automated actors.

---

## Implementation Difficulty and User Impact

The following table ranks each class of detection signal by how difficult it is to implement, whether it requires user involvement, and its likely uniqueness and value as a behavioral or platform identifier.

| Detection Category                   | Implementation Difficulty | User Involvement | Signal Strength | Notes |
|-------------------------------------|----------------------------|------------------|------------------|-------|
| JavaScript Engine Quirks            | Low                        | None             | High             | Works reliably even on older devices |
| Rendering & Font Fidelity           | Medium                     | None             | High             | Needs normalization across OS/Display |
| Subpixel Rendering & Display Fidelity | High                    | None             | High             | Detects physical monitor and compositor stack quirks |
| Behavior and Movement Fingerprinting| Medium                     | Passive          | High             | Training data improves accuracy |
| Timing & System Clock Checks        | Low                        | None             | Medium           | Clock skew + drift are good flags |
| Storage and Permission Behavior     | Low                        | None             | Medium           | May be falsified in hardened VMs |
| Network & Prefetch Behavior         | Medium                     | None             | Medium           | Useful for edge latency profiling |
| Platform Consistency Clues          | Low                        | None             | Medium           | Easy to test but easy to spoof |
| Locale & Calendar Realism           | Low                        | None             | Medium           | Works well in tandem with IP location |
| DOM and Isolation Fidelity          | Medium                     | None             | High             | DOMParser & iframe quirks are robust |
| Interactivity Coherence             | Medium                     | None             | High             | Requires timed capture of focus/mouse |
| OS Integration Hints                | Medium                     | None             | Medium           | Clipboard and TouchEvent fields help |
| Accessibility Behavior              | Low                        | None             | Low              | Weak alone but good auxiliary data |
| Environmental Latency Signatures    | High                       | None             | High             | GPU and event jitter tests shine here |
| Traditional CAPTCHA (baseline)      | Low                        | High             | Medium           | Works best against unsophisticated bots |

This matrix helps guide which categories are valuable early candidates for deployment or further training and which might need more robust infrastructure or passive fallback implementations.

---

## Inspirations and Related Work

- [FingerprintJS](https://github.com/fingerprintjs/fingerprintjs)
- puppeteer-extra-plugin-stealth
- FPMON: Performance-based browser fingerprinting
- FPSCanner: Browser inconsistencies for headless detection
- Cloudflare Bot Management and JA3 fingerprinting

---

## Roadmap

- Add sample ONNX model for feature vectors
- Collect dataset of real + bot sessions for tuning
- Expand `chore_registry.json` and dynamic chore bundling
- Visual annotation for audit verdicts
- Optional LLM integration for forensic explanations

---

### 🧾 CSS Feature and Cascade Fidelity

Spoofed environments often fail to fully implement advanced CSS capabilities or cascade behavior. BluffCharge includes tests that verify real rendering and style resolution rather than relying on `CSS.supports()` claims.

**Examples:**

- `:has()` Selector — Valid in most evergreen browsers; not polyfillable.
- `@container` Queries — Layout response to container query conditions.
- CSS `initial-letter` — Detect dropped-cap typography support.
- Shadow DOM Style Leakage — Validate encapsulation rules with scoped CSS.
- `display: contents` — Render an invisible wrapper and test box model collapse.
- `text-wrap: balance` — Visual verification of new text layout properties.

These tests are rendered offscreen and inspected via layout measurements and bounding rects.

---

### 🧪 WebAssembly Support Surface

WASM discrepancies can expose runtime emulation and browser engine inconsistencies. We test both support and behavior:

**Tests:**

- `WebAssembly.validate()` with edge-case binaries
- `compile()` vs `compileStreaming()` timing comparison
- Structured cloning of WASM modules
- Memory aliasing detection across contexts
- Streaming import error message validation

---

### 🧩 DOM Edge Quirks

Falsified clients or headless engines often mishandle rare or ambiguous DOM behaviors:

- Detached tree insertions
- Unclosed HTML fragment handling via `innerHTML`
- Event retargeting and `composedPath()` checks across Shadow DOM
- Text node reflow behavior under mutation

---

### 📉 Math Precision Drift

Floating-point math reveals subtle implementation differences:

- Iterative derivation of π and `e` with bitwise accuracy checks
- Tests using `Math.hypot`, `Math.acosh`, `Math.expm1` with drift measurement
- Rounding behavior of `Float32Array` and `Float64Array` at edge inputs

---

### 🚫 Security Policy Handling

Security-focused behavior is hard to emulate perfectly:

- `eval()` blocked under simulated CSP environments
- iframe sandbox flag effects on focus and form submission
- Cross-origin iframe JavaScript isolation tests

---

### 🕸️ Network Cache and Freshness Drift

Legitimate browsers show realistic cache behavior that’s difficult to spoof:

- `Cache-Control: no-cache` handling consistency
- Stale-While-Revalidate fetch delay testing
- ETag-based conditional GET behavior
- Response freshness timing for ServiceWorker-mediated requests

### 🔤 Focus Ring and Tab Fidelity

Some automation frameworks simulate navigation via direct `.focus()` calls rather than using real keyboard traversal. This can be detected by:

- Observing `Tab` key events and natural focus order.
- Monitoring for native focus ring rendering via CSS `:focus-visible`.
- Tracking `focusin`/`focusout` without corresponding key events.

These tests are non-intrusive and expose discrepancies in synthetic input flows.

---

### ⌨️ Synthetic Key Event Timing

Human typing introduces natural delay variation between keystrokes. Bots and scripted input often exhibit:

- Fixed inter-keystroke intervals.
- Lack of jitter between down/up sequences.
- Unnatural burst typing patterns.

We log typing entropy for fields like login or form completion and flag suspicious uniformity.

---

### 🌐 WebRTC Connection Entropy & Timing

Beyond ICE candidate content, we measure:

- Time to gather ICE candidates (delays reveal TURN fallback).
- Protocol variation (`udp`, `tcp`, `relay`, `host`, `srflx`).
- Failure to resolve mDNS hostnames (inaccurate emulation).

Establishing peer connections with internal STUN servers enables reproducibility and NAT fingerprinting.

---

### 🌑 Shadow DOM Retargeting & Isolation

Even in browsers with Shadow DOM support, retargeting inconsistencies can leak emulation:

- Compare `event.target` and `event.composedPath()` in deep trees.
- Monitor event bubbling across open/closed roots.
- Validate scoped style encapsulation vs fallback polyfill leakage.

These microbehaviors are difficult to emulate precisely.

---

### 🖥️ OS-Level Customization Propagation

Detects whether system-wide settings affect rendering:

- Font size scaling via accessibility settings.
- Scrollbar thickness changes.
- Forced high-contrast or inverted colors.

Bots rarely replicate these, and headless modes often apply defaults.

---

### 🕰️ Replay Drift Detection

To prevent replay of chore responses:

- Chore requests include heartbeats with server-seeded time.
- Timing of response arrival vs expected phase is analyzed.
- Some tasks depend on prior state (inter-chore correlation).

Late or reordered replies reveal tampering or bot orchestration artifacts.

---

### 🎨 GPU Thread Scheduling Drift

True GPU pipelines introduce variance in:

- `canvas.measureText()` rendering under animation.
- `requestAnimationFrame` vs `setTimeout` drift patterns.
- Canvas redraws with gradients or transformations.

Synthetic renderers often skip over multi-threaded latency, producing “too clean” samples.

---

### ✋ Touch vs Pointer Drift

On real touchscreens:

- `touchmove` paths differ from `mousemove`.
- Event velocity curves exhibit physical finger constraints.
- Zoom/pinch inertia follows platform-specific physics.

Synthetic input often blends both or produces linear motion.

---

### 🔋 Battery API Presence and Consistency

Battery API presence is optional but revealing:

- `navigator.getBattery()` often undefined in bot contexts.
- Charging rate, level, and discharge time should match device class.
- Repeated queries should show realistic slow variation.

Absence or staleness can indicate VMs, headless stacks, or misreporting.

---

### 🧊 Zero-Entropy Signal Suspicion

Too-consistent timing is a red flag:

- Identical `requestIdleCallback` delay cycles.
- `mouse.move` paths without jitter or noise.
- Repaint timing perfectly uniform under animation stress.

Entropy floor violations trigger suspicion scores in ONNX audit logic.

---

### 🐛 DevTools Detection

Passive devtools signals include:

- Delays introduced by breakpoints or console inspection.
- Overridden `Function.prototype.toString()` behavior.
- Timing of `console.log()` affected by open console.
- `debugger` keyword silently delaying execution.

Such tests are silent, cheap, and reveal local probing or inspection attempts.
### 🖋️ Font Rendering Fidelity

Subpixel smoothing and rasterization differ by OS and are hard to spoof.

- Render known font glyphs (e.g. "Ag") onto a canvas.
- Zoom in on edges to detect RGB subpixel patterns or grayscale hinting.
- Compare pixel color variance across vertical and horizontal edges.
- macOS typically uses grayscale; Windows uses ClearType (subpixel RGB).
- Linux environments often reveal no subpixel alignment at all.

Used to verify native rendering stack consistency with claimed platform.

---

### 🕶️ Legacy API Exposure (VRML, WebXR)

Legacy APIs, while rarely used, serve as detection surface:

- Check for existence of `navigator.xr`, `navigator.vr`, or VRML remnants.
- Attempt partial instantiation of `WebXRDeviceRequest` or similar.
- Look for `document.createVRDisplay()` in older browser engines.

Absence of these APIs under “evergreen” User-Agents indicates tampering.

---

### ⌛ Boot & Session Drift

Measure differences between system session origin and browser-reported timing:

- Compare `performance.timeOrigin` with server navigation log.
- Analyze inconsistency in `performance.now()` vs wall clock time.
- Detect repeated resets of origin — common in ephemeral VMs or replays.

This reveals disposable automation clients.

---

### 🏷️ OS Lie via Font Probe

Detect false claims about operating system via:

- `document.fonts.check()` against OS-unique fonts:
  - `-apple-system`, `.SF NS`, `.SF Pro` → macOS
  - `Segoe UI` → Windows
  - `Noto Sans` → Android
- Absence of platform fonts despite claimed OS = lie signal.

Used in combination with raster tests for strong confirmation.

---

### 🧮 Platform Signal Crosscheck

Cross-validate internal `navigator` fields:

- Do `navigator.deviceMemory`, `hardwareConcurrency`, `maxTouchPoints`, and `userAgent` align?
- Is `touchstart` present if touch points > 0?
- Do screen dimensions match device class (e.g., touch phone resolution with no touch events)?

Mismatch implies spoofing or proxying via misconfigured automation.

---

### 💾 Storage Mechanism Fidelity

Test for inconsistencies or stubbed APIs:

- File System Access API presence but no read/write permission.
- `navigator.cookieEnabled` true but `document.cookie` always empty.
- Huge `navigator.storage.estimate().quota` despite small RAM footprint.

These reveal faked or overly permissive storage models.

---

### ✍️ Editable DOM Fidelity

Script-based clients often skip edge DOM tests like:

- `contenteditable` caret tracking via `selectionchange`.
- `execCommand('bold')` behavior and undo stack.
- Insertion of styled elements via paste.

Capture fidelity drift by auto-inserting and inspecting resulting DOM structure.

---

### 🔊 Audio Engine Divergence

`AudioContext` responses differ per browser/OS/hardware:

- Create oscillator, render wave to buffer, inspect `getChannelData()` drift.
- Validate latency output from `baseLatency` or `outputLatency`.
- Compare timing of `start()` and actual audio fire if possible.


---

### 📱 Hybrid Input Contradiction Detection

Discrepancies between reported input capabilities and real-world interaction can indicate spoofed or headless clients.

- Check `navigator.maxTouchPoints` against presence of `ontouchstart`, `PointerEvent.pointerType`, and actual gesture events.
- Look for mouse-only behavior on devices claiming multitouch capability.
- Test for platform-level scrolling physics, like inertial momentum and rubber-banding, on touch-capable devices.
- Analyze hover behavior — touchscreens do not emit consistent `mouseover` sequences.

These inconsistencies help flag environments that falsify mobile or tablet characteristics.

---

### 🎯 DeviceMotion and Gyroscope Variance

In environments that support it, motion sensors can reveal emulated or static states.

- Use `DeviceMotionEvent` and `DeviceOrientationEvent` to log accelerometer and gyroscope data.
- Real devices exhibit micro-variation even at rest due to gravity and minor movement.
- Bots and emulators often provide flat, zeroed, or constant output.

**Caveats:**
- Modern browsers require HTTPS and sometimes a user gesture before granting sensor access.
- Recommended to defer this test until a user scroll or tap occurs.

Collected data is included in ONNX appraisal and timestamped for cross-sensor coherence checks.
### 🔁 Conditional Page Reload Challenge

In high-suspicion contexts or post-authenticated sessions, a controlled page reload can validate environment consistency and reveal automation limits.

**Purpose:**
- Confirm session continuity
- Detect improper persistence or stateless replay
- Observe event dispatch behavior under real navigation lifecycle

**Strategies:**

- `window.location.reload(true)`  
  Forces a hard reload bypassing cache. On reload, validate sessionStorage presence, scroll restoration, and interaction history.

- `location.replace(currentURL)`  
  Performs a soft refresh without history entry. Measure timestamp coherence and DOM hydration integrity.

- `meta[http-equiv=refresh]`  
  Injected dynamically after client passes a suspicion threshold. Reflow behavior and media query reevaluation can be measured post-reload.

- `beforeunload` + `pagehide` tracking  
  Attach state-preserving or signaling code before navigation loss. On reload, compare hash, page state, or entropy vectors to previous instance.

**Usage Guidelines:**
- Only deploy in confirmed non-human scenarios or in logged-in, low-risk conditions
- Do not perform reloads repeatedly — one-time trigger tied to audit state
- Log client-reported state alongside server-held session fingerprint for discrepancy detection

The refresh challenge is not a user-facing captcha, but an invisible behavioral integrity test.