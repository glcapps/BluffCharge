# BluffCharge
Web Browser and Visitor Sincerity Testing

## Overview

BluffCharge is a research and detection framework designed to evaluate the sincerity of web visitors. It performs passive, dynamic, and auditable testing of browser environments to detect spoofing, automation, and evasive behavior.

## Key Concepts

- **Self-Appraisal with Audit Trail**: The client performs self-assessment using designated detection tasks and submits both results and a hash of inputs seeded by a server heartbeat, preventing retroactive falsification.
- **ONNX Model Appraisal**: Client results are evaluated by an embedded ONNX model to determine environmental plausibility and behavior authenticity.
- **Passive and Non-Invasive**: Designed to run tests that do not interrupt the user experience, unlike traditional CAPTCHA systems.

- **Retrospective Verification**: Client self-appraisal submissions are later re-evaluated server-side using the original input log and a server-issued heartbeat seed, revealing attempts to falsify sincerity assessments.

## Detection Categories

### JavaScript Engine Quirk Detection
- Syntax feature support (e.g., optional chaining, private fields)
- Syntax error message structure
- Eval and scoping edge cases
- Native function introspection
- Microtask timing irregularities

### Browser and OS Environment Tests
- Font rendering and subpixel pattern analysis
- Scroll and pointer movement entropy
- Canvas rendering differences
- AudioContext and WebGL jitter
- Notification and permissions behavior
 - Touch capability vs pointer behavior mismatch
 - Scroll momentum presence on touch-claimed devices
 - SpeechSynthesis and SpeechRecognition availability
 - Clipboard metadata structure on passive paste
 - Push API and Notification permission realism

### Timing and Network Behavior
- Beacon delivery delay under load
- WebSocket echo latency drift
- Artificial 204 latency reflection
- Prefetch trap sequencing

### Storage and Persistence Consistency
- Local/sessionStorage across reloads
- IndexedDB behavior with extreme versions
- navigator.storage quota estimation

### Locale and Time Fidelity
- Clock skew from server time
- Timezone offset vs. geolocation
- Intl date formatting locale mismatches
- First day of week rendering in calendar

### Passive Interactivity Fingerprinting
- Visibility change and tab switching
- Mouse drift patterns
- Selection entropy and caret behavior
 - Inertial scroll detection
 - Gyroscope and accelerometer jitter analysis (deferred until first gesture)

### DOM and Style Rendering Fidelity
- MutationObserver delay under idle
- Scrollbar dimension tests
- Iframe sandbox leakage probes
- getComputedStyle default order checks

### Plugin and MimeType Cross-validation
- navigator.plugins vs. mimeTypes consistency
- navigator.webdriver and headless flags
- navigator.hardwareConcurrency realism

### Microbenchmark Jitter and Noise
- JSON.parse timing consistency
- DOM layout reflow jitter
- Array sort performance skew

## Design Innovations

- **Audit-Seeded Logging**: Each self-appraisal record is linked to a prior heartbeat with a server-side timestamp to prevent tampering.
- **Client-side Chore Modules**: Tasks are modular and dispatched dynamically via a `chore_registry.json`.
- **Backfill Detection**: The audit model can request re-evaluation of prior periods, making it hard to falsify earlier logs.
- **Low-Friction vs CAPTCHA**: Compared to traditional visual CAPTCHA tasks, BluffCharge achieves similar fraud resistance with lower user friction.
- **Trust-Adaptive Task Dispatch**: Detection tasks escalate in depth or intensity based on client trust level, allowing minimal friction for sincere users and deeper validation for edge cases.

## Audit Architecture

BluffCharge uses a heartbeat-seeded audit system:

- Each self-appraisal is timestamped and linked to a periodic heartbeat issued by the server.
- The client computes a hash of their environmental input and appraisal result using the heartbeat as part of the seed.
- These submissions are stored and later re-evaluated on the server using the same heartbeat, revealing any mismatched or falsified self-assessments.
- The audit engine supports revalidation of earlier activity logs, discouraging attempts to backfill falsified session data.

## Future Roadmap

- Visual fingerprint collection helper for canvas and font tests
- ONNX model training using high-fidelity real user session data
- Additional DOM isolation and accessibility tests
- Experimental support for latency-based monitor inference
- Dynamic server-chore assignment with expiration logic
  - Conditional refresh challenges for high-suspicion sessions

## Related Work

- FingerprintJS (open source & pro)
- puppeteer-extra-plugin-stealth
- FPSCanner by Antoine Vastel
- FPMON: Microbenchmark fingerprinting
- Cloudflare Bot Management
- Kasada and Arkose Labs
