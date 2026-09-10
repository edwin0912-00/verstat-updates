# Yellow Code Protocol: Autonomous Agent Operating Rules

> **Status:** ACTIVE & MANDATORY FOR ALL AGENTS  
> **Transition:** Red Code (Emergency Hotfixing) ➔ Yellow Code (Disciplined Step-by-Step Delivery)  
> **Scope:** Fleet Dispatchers, Codex Developers, Antigravity Agents, and CI/CD Automation

---

## 1. Context & Motivation

Emergency hotfixing ("Red Code") has completed. The auto-update feed architecture, cryptographic signatures, SSH sync keys, and base build invariants have been stabilized.

We are now operating under **Yellow Code**:
- No reckless rushing.
- Zero speculative complexity.
- Every modification must be deliberate, measured, and verified before online publication.
- Every agent touching this repository or the `video-variation-studio` codebase must comply with this protocol without exception.

---

## 2. The 3 Core Mandates of Yellow Code

### Mandate 1: Release & Delta Handshake (Strict Human Approval)
Autonomous agents MUST NOT build, package, generate deltas, or publish updates online automatically.

Before generating any release or delta, the agent must pause and present a 4-point handshake:
1. **Target Build Version**: (e.g., `Build 346`)
2. **Delta Reference Baseline**: (e.g., strictly `345 ➔ 346`)
3. **Payload Estimates**: Expected delta size (~500 KB) and full package size (~1.6 GB).
4. **Deployment Decision**: Explicit question whether to deploy online immediately or stage locally for manual verification.

---

### Mandate 2: Bug Prioritization & Transparency Matrix
Whenever an issue, defect, or bug is reported, agents must present a structured assessment before writing code:
1. **Plain-Language Technical Explanation**: What the defect actually is, stripped of jargon.
2. **Estimated Resolution Time**: Realistic estimate in minutes.
3. **Exact Code Location**: File path, component, and architectural layer.
4. **Permanence Guarantee**: Architecture-level resolution vs. temporary band-aid.
5. **Priority Classification**:
   - **P0 (Critical / Blocker)**: Crash, data loss, runtime failure, broken build.
   - **P1 (High / Functional & UX)**: Layout defects, button text wrapping, broken workflow, sync failure.
   - **P2 (Normal / Polish)**: Visual inconsistencies, non-blocking telemetry, minor wording.

Work proceeds only after the operator agrees with the priority and approach.

---

### Mandate 3: Pre-Mortem & UX Impact Forecasting («Пророкування грабель»)
Before touching any UI component, button, or adding new features (such as object tracking, timeline controls, modal inspectors):

1. **System Stability & Resource Forecasting**:
   - Does the change introduce new dependencies? (Zero external dependencies invariant).
   - Will background processing (e.g., tracking, rendering, audio isolation) starve the main UI thread?
   - What is the memory and GPU overhead during batch execution?
2. **UX & Responsive Layout Constraints**:
   - How does the layout behave at the minimum supported window size (`960×620`)?
   - Are action buttons protected against word-wrapping? Enforce `.lineLimit(1)` and `.fixedSize(horizontal: true, vertical: false)`.
   - Does changing or moving a button disrupt existing user muscle memory?
3. **Data & State Compatibility**:
   - Will existing saved sessions, profiles, and export presets deserialise without errors?

---

## 3. Sparkle Update Gate Zero Invariants (Uncompromising)

Every agent modifying updates, deltas, or `appcast.xml` must enforce:
- **Invariant 0**: `<sparkle:deltas>` MUST be a direct child of `<item>` (sibling to `<enclosure ... />`), NEVER nested inside `<enclosure>`.
- **Invariant 1**: Zero `<sparkle:channel>` tags in public/dev-alpha feeds.
- **Invariant 2**: Explicit Keychain account `sign_update --account variator_astra_dev_alpha` for feed XML and all binaries.
- **Invariant 3**: Standalone release package only (>= 1.2 GB uncompressed, node >= 50 MB, zero dev-runtime contamination).
- **Invariant 4**: Local `BinaryDelta apply` & `codesign --verify` verification pass on baseline before release.
- **Invariant 5**: Automated pre-flight gate execution via `./script/verify_sparkle_feed.sh`.

---

## 4. Compliance & Verification

Any pull request, commit, or automated branch that violates these mandates will be rejected by the automated feed gate and Wiser regression rules (`R-005`).
