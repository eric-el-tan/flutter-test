# Implementation Plan: Local-First PDF Redactor (Flutter)

Built from [.claude/project-overview.md](project-overview.md) plus explicit decisions gathered
via clarifying questions (recorded below). This plan supersedes the overview's roadmap where
the two disagree — the decisions below are authoritative.

---

## 0. Confirmed Decisions (source of truth)

| Area | Decision |
|---|---|
| State management | **Riverpod** (not BLoC, not plain Provider) |
| PDF library | **Syncfusion** (`syncfusion_flutter_pdfviewer` + `syncfusion_flutter_pdf`), free **Community License** (confirmed solo-project eligible) |
| Redaction technique | **True content-stream text removal** — strip the underlying text-showing operators for matched strings, then draw an opaque black box. Not rasterization. |
| Platforms (this MVP pass) | **Android only.** iOS explicitly deferred — no iOS build/TestFlight tasks in this plan. |
| Min SDK | `minSdkVersion 21` (Android 5.0) |
| Test device | Emulator only (no physical device available) |
| Flutter SDK | **Not yet installed.** Install latest **stable** channel, compatible with Syncfusion/Riverpod current versions. Installation is Phase 0, not assumed. |
| PII pattern scope | Email, phone (**NZ format**), **NZ IRD number**, generic numeric ID, custom regex/keyword search. **Simple regex shape-matching only** — no checksum/check-digit validation, no complex validators. Accepting more false positives is the deliberate tradeoff; the pattern-chain structure stays in place so validators can be added later without rework. |
| OCR (scanned PDFs) | **Out of scope for MVP.** Image-only PDFs are detected and rejected with a clear message. Post-MVP task. |
| Trial gate | View and scan **any** document freely, regardless of page count. **Export** ("Redact & Save") is blocked on documents **>3 pages** unless purchased. |
| Redacted output location | **New file** alongside original, app-local storage via `path_provider`, `<original>_redacted.pdf` naming. Original is never modified. |
| Redaction review UX | **Both flows built**, user picks via a **Settings toggle**: (a) manual review — tap to include/exclude each match before applying, or (b) auto-redact-all matches on scan |
| Architecture rigor | **Lightweight/pragmatic**, not full Clean Architecture from `flutter_project_guidelines.md`. Feature-based folders + UI/logic separation kept; GetIt, use-case classes, repository interfaces, and freezed are **skipped** for MVP. Riverpod providers call services directly. |
| Testing scope | Minimal, targeted: unit tests for the PII pattern/validator chain and the redaction coordinate math only. No widget/integration test pyramid for MVP. |
| App identity | **Placeholder** — name `PDF Redactor`, package `com.example.pdf_redactor`. Renaming to real identity is an explicit pre-launch task, not done now. |
| Store/monetization accounts | **Out of scope for this pass — app first.** No Google Play Console account creation, no RevenueCat project setup, no real store product configuration. Week 3 builds the trial gate and paywall **UI** against a local unlock flag (stubbed purchase). Wiring real `purchases_flutter` + store products is a deferred post-MVP task. |
| Local storage scope | **Settings only** via `shared_preferences`: manual-review-vs-auto-redact toggle, default-enabled PII categories, trial/purchase status. No history list, no saved rule templates in MVP. |
| Encrypted/corrupt PDFs | **Detected and rejected** with a clear in-app error message — not left to crash. |
| Time budget | **3-week calendar held**, but weekly hour estimates are **expanded to realistic levels** rather than forced to fit the original 18h. See per-week totals below (~28.5h, not 18h) — this reflects the real scope of the decisions above (Riverpod, true content-stream redaction, dual review UX, from-scratch tooling), not padding. |

**Assumptions I'm making that you should sanity-check** (flag if wrong, I'll adjust):
- "Latest stable Flutter" as of project start will be checked live via `flutter --version` after install, not pinned to a number in this doc.
- Exact Syncfusion API surface for content-stream text removal will be verified against current Syncfusion Flutter PDF docs at the start of Week 2 (a short spike, budgeted below) — the overview's description of the capability is directionally correct but I'm not asserting exact method names without checking current docs first, since Syncfusion's API has changed across versions.
- NZ phone and NZ IRD are matched by **shape only** — plain regex, no check-digit math — per your instruction to skip complex calculation for now. Exact patterns are in §3 below for your review before I implement them.

---

## 1. Phase 0 — Environment Setup (prerequisite, before Week 1 hours start)

Not part of the 18/30-hour coding budget — this is one-time machine setup, budgeted separately.

| Task | Est. |
|---|---|
| Install Flutter SDK (stable channel) | 0.5h |
| Install Android Studio + Android SDK + create an AVD emulator (API 21 image, or a newer image is fine since minSdk 21 just sets the floor) | 0.75h |
| Run `flutter doctor`, resolve any Android toolchain/license issues (`flutter doctor --android-licenses`) | 0.5h |
| Install VS Code Flutter/Dart extensions (already using VS Code per environment) | 0.1h |
| Register a Syncfusion account, confirm Community License eligibility in their portal, generate a license key | 0.5h |
| `flutter create pdf_redactor` scaffold, confirm it builds and runs on the emulator (default counter app) | 0.25h |

**Phase 0 total: ~2.5h.** Gate: emulator boots the default Flutter counter app before Week 1 Task 1.1 starts.

---

## 2. Week 1 — Project Scaffolding & Local PDF Rendering

**Goal:** Open a local PDF and render it smoothly on Android, on the real architecture (Riverpod,
feature folders), not a throwaway prototype. **Revised budget: ~8h** (was 6h).

### Task 1.1 — Dependencies, Riverpod Bootstrap & Folder Structure (2h)
- Add to `pubspec.yaml`: `syncfusion_flutter_pdfviewer`, `syncfusion_flutter_pdf`,
  `file_picker`, `path_provider`, `flutter_riverpod`, `shared_preferences`, `share_plus`
  (share task is Week 3 but the dependency can go in now).
- Register the Syncfusion license key (`SyncfusionLicense.registerLicense(...)`) in `main.dart`
  before `runApp`.
- Wrap the app root in `ProviderScope`.
- Create the lightweight feature-based structure:
  ```
  lib/
  ├── app/
  │   ├── app.dart              # MaterialApp + theme + routes
  │   └── theme.dart
  ├── core/
  │   ├── errors/                # PdfLoadException, EncryptedPdfException, CorruptPdfException
  │   └── utils/
  ├── features/
  │   ├── document/
  │   │   ├── presentation/      # HomeScreen, ViewerScreen
  │   │   └── document_providers.dart   # Riverpod providers/notifiers + service fns, no separate data/domain split
  │   ├── pii_scan/               # (Week 2)
  │   ├── redaction/              # (Week 2)
  │   ├── settings/               # (Week 3)
  │   └── paywall/                # (Week 3)
  └── main.dart
  ```
- **Acceptance:** `flutter run` launches to an empty Home screen with no analyzer warnings.

### Task 1.2 — File Picker & Document Loader (2h)
- Home screen with an "Open Document" button using `file_picker` (`FileType.custom`,
  `allowedExtensions: ['pdf']`).
- Handle Android storage/media permission prompts (Android 13+ uses scoped media permissions;
  `file_picker`'s SAF-based picker avoids needing broad storage permission — confirm this holds
  at minSdk 21 too, since older Android may need `READ_EXTERNAL_STORAGE`).
- Load the picked file into a Riverpod `AsyncNotifier` holding the current document path/bytes.
- **Error handling (per decision):** attempt to open via Syncfusion's `PdfDocument`; catch load
  failures and classify as encrypted vs corrupt vs unknown, surfacing a specific in-app message
  for each (not a generic crash).
- **Acceptance:** picking a valid PDF navigates to the viewer; picking an encrypted or corrupt
  PDF shows the correct specific error state without crashing.

### Task 1.3 — PDF Viewer Screen (2.5h)
- Viewer screen using `SfPdfViewer.file()`, bound to the loaded document from 1.2.
- Verify page navigation, pinch-zoom, and correct rendering across a multi-page test PDF.
- Capture and expose `pageCount` from the loaded document via the provider (needed by both the
  trial gate in Week 3 and the PII scan flow in Week 2).
- **Acceptance:** open a 10+ page real-world PDF (mixed text sizes/orientation), scroll and zoom
  through all pages without jank or crash on the emulator.

### Task 1.4 — Manual Smoke Test Pass (1.5h)
Not in the original roadmap, but needed since minSdk 21 + emulator-only testing is the whole
verification surface for this project — no physical device to catch device-specific issues.
- Test with: a small (1–2 page) text PDF, a large (20+ page) text PDF, a PDF with mixed
  fonts/embedded images, and a scanned/image-only PDF (confirm it loads for *viewing* even
  though PII scanning will later reject it per the OCR-out-of-scope decision).
- Log any Syncfusion license-key or rendering issues now, before building on top of it in Week 2.

**Week 1 total: ~8h.** **Deliverable:** Pick any local Android-accessible PDF, view all pages
reliably; encrypted/corrupt files fail gracefully; architecture skeleton is in place for Week 2.

---

## 3. Week 2 — PII Detection & Local Redaction Engine

**Goal:** Detect PII via a verifiable pattern chain, let the user review or auto-apply matches
per their settings choice, and truly strip matched text from the PDF (not just paint over it).
**Revised budget: ~12h** (was 6h) — this is the technically hardest week given the true
content-stream removal requirement and dual review UX.

### Task 2.1 — Syncfusion Redaction API Spike (1h) — *do this first*
- Before committing to an implementation approach, verify against **current** Syncfusion
  Flutter PDF documentation what API actually exists today for (a) locating text run
  coordinates via `PdfTextExtractor`, and (b) removing/clearing the underlying text-showing
  operators for a matched string from a loaded page's content stream (as opposed to just
  overlaying a shape). Syncfusion's PDF package has changed across versions; confirm exact
  class/method names before Task 2.4 depends on them.
- **Output:** a one-paragraph note (in code comments or a scratch file) confirming the approach,
  or flagging a fallback if true content-stream removal isn't cleanly exposed and a documented
  workaround (e.g., re-writing the page's content stream manually) is required instead.

### Task 2.2 — PII Pattern Chain Engine (1.5h)
Design as a chain of typed pattern matchers, not one flat regex list. The `validate` hook stays
in the model so checksum validators can be dropped in later, but **every MVP pattern ships with
`validate: null`** — simple regex shape-matching only, per your instruction:

```dart
class PiiPatternType {
  final String id;              // 'email' | 'nzPhone' | 'nzIrd' | 'numericId' | 'custom'
  final String label;
  final RegExp candidateRegex;
  final bool Function(String match)? validate; // null for ALL MVP patterns; hook for later
}
```

Proposed patterns — simple, no calculation (**flag if any should change before I implement**):
- **Email:** practical email regex, e.g. `RegExp(r'\b[\w.+-]+@[\w-]+\.[\w.-]+\b')`.
- **NZ phone:** shape match on `+64` or `0`-prefixed numbers with optional spaces/hyphens, e.g.
  `RegExp(r'(?:\+64|0)[\s-]?\d(?:[\s-]?\d){7,9}')`. No mobile-vs-landline digit-count check.
- **NZ IRD number:** shape match on 8–9 digit grouped numbers, e.g.
  `RegExp(r'\b\d{2,3}[-\s]?\d{3}[-\s]?\d{3}\b')`. **No check-digit validation.**
- **Generic numeric ID:** run of 6+ digits, best-effort.
- **Custom:** user-entered literal keyword or raw regex, used as-is.

- **Known tradeoff:** without checksums, the IRD and generic-numeric patterns will match
  unrelated long numbers (invoice numbers, dates, reference codes). This is why the manual-review
  UX flow from Task 2.3 matters — it's the user's false-positive escape hatch. Worth considering
  making manual review the *default* setting because of this.
- **Acceptance:** unit tests (per the "minimal targeted tests" decision) covering true positives
  per pattern type and overlapping-match de-duplication.

### Task 2.3 — Text Extraction, Match Positioning & Highlight Overlay (3h)
- Use `PdfTextExtractor` to pull page text plus bounding rectangles.
- Run the pattern chain from 2.2 against extracted text, mapping matches back to on-page
  rectangles.
- Overlay highlight boxes on the `SfPdfViewer` for matched regions.
- **Both review UX flows, gated by the Settings toggle (decided, not deferred):**
  - **Manual review:** matches render as tappable highlight boxes the user can
    include/exclude; a running count of "N of M matches selected" and a "Redact Selected"
    action.
  - **Auto-redact-all:** matches render as highlights for visibility only; "Redact & Save"
    applies to every match found, no per-match toggle needed.
- **Acceptance:** scanning a test PDF with known planted PII (email/phone/IRD/keyword) surfaces
  all of them as highlights in both UX modes, with the manual mode's include/exclude state
  correctly tracked in a Riverpod notifier.

### Task 2.4 — Redaction Execution (True Content-Stream Removal) (4h)
The core technical risk of the whole project — budgeted accordingly.
- For each confirmed match: locate and remove/clear the text-showing operator(s) responsible
  for that string in the page's content stream (per the Task 2.1 spike's confirmed approach),
  then draw an opaque black rectangle over the same bounds.
- **Verification step (not optional):** after redaction, re-run `PdfTextExtractor` on the
  output and assert the redacted string no longer appears anywhere in extracted text — this is
  the actual proof the redaction worked, not just "a black box is visually present."
- Handle edge cases: a match spanning a line wrap, overlapping matches from different pattern
  types on the same text run.
- **Acceptance:** for every planted-PII test PDF from 2.3, the exported file's extracted text
  contains **zero** occurrences of the redacted strings, and the visual black box is correctly
  positioned.

### Task 2.5 — Save Redacted Output (1.5h)
- Save via `path_provider` to app-local storage as `<original-filename>_redacted.pdf`,
  alongside (not replacing) the original, per the output-handling decision.
- Handle filename collisions (e.g., re-redacting the same source twice) with a numeric suffix.
- **Acceptance:** redacted file appears at the expected app-local path and reopens correctly in
  the viewer.

### Task 2.6 — Encrypted/Corrupt Detection Hardening (1h)
- Extend Task 1.2's error handling to the redaction path specifically — confirm a document that
  loads for viewing but fails during text extraction (rare malformed PDFs) also fails gracefully
  rather than mid-redaction.

**Week 2 total: ~12h.** **Deliverable:** Scanning a document (any page count) highlights PII per
the pattern chain; user reviews (per their settings choice) and exports; the exported PDF has
matched text verifiably unextractable, saved as a new file next to the original.

---

## 4. Week 3 — Trial Gate, Monetization, Settings, Polish & Build

**Goal:** Gate exports over 3 pages, add the settings that Week 2 depends on, and produce a
verified local Android build. **Revised budget: ~6h.** No store accounts are created in this
pass — the app itself is the target.

### Task 3.1 — Settings Screen & Local Storage (1.5h)
- `shared_preferences`-backed settings provider storing (per the local-storage-scope decision,
  settings only):
  - Manual-review vs auto-redact-all toggle (read by Task 2.3's UX branch)
  - Default-enabled PII pattern categories (email/phone/IRD/custom on/off)
  - Purchase/unlock status flag (written by Task 3.2)
- Simple settings screen wiring these to the provider.
- **Acceptance:** toggling manual-review in Settings changes the scan screen's behavior on next
  scan without app restart.

### Task 3.2 — Free Trial Gate (1h)
- Read the loaded document's `pageCount` (already exposed from Week 1 Task 1.3).
- Per the trial-gate decision: viewing and scanning are **never** blocked. Only the
  "Redact & Save" export action checks `pageCount > 3 && !purchased` and, if true, opens the
  paywall instead of exporting.
- **Acceptance:** a 4-page doc scans and highlights normally; tapping export opens the paywall
  instead of saving. A 3-page-or-fewer doc exports normally regardless of purchase state.

### Task 3.3 — Paywall UI with Stubbed Purchase (1.5h)
- Build the paywall bottom sheet (single $4.99 USD one-time unlock, copy and layout final),
  triggered from Task 3.2's gate and from a manual "Unlock Full Version" entry point.
- Behind a single `PurchaseService` interface with one method (`Future<bool> purchaseUnlock()`),
  implemented for now by a **stub** that just writes the unlock flag from Task 3.1. This keeps
  the real `purchases_flutter` implementation a drop-in swap later, with no UI rework.
- **No** `purchases_flutter` dependency, RevenueCat project, or store product in this pass.
- **Acceptance:** tapping unlock in the paywall enables export for >3-page documents, and the
  unlock persists across app restarts. A debug-only "reset unlock" action makes the gate
  re-testable.

**Deferred (post-MVP, once store accounts exist):** create Google Play Console account, create
RevenueCat project, configure the non-consumable product, swap the stub for the real
`purchases_flutter` implementation, and validate a test-track purchase.

### Task 3.4 — Share, Dark Mode, UI Polish (1.5h)
- "Share Sanitized PDF" via `share_plus` on the redacted output from Week 2 Task 2.5.
- Dark mode theming.
- Loading indicators during scan/redaction (Task 2.4 is not instant on larger documents).
- Basic empty/error states polish for the flows built in Weeks 1–2.

### Task 3.5 — Build Verification & Test Run (1.5h)
- `flutter build apk --release` (and `flutter build appbundle` to confirm it produces cleanly),
  install and run on the emulator.
- Full manual regression pass: pick → view → scan (both UX modes) → redact → verify unextractable
  text → export/share → trial gate on a >3-page doc → stubbed unlock → export succeeds.
- **Confirm zero network traffic** across the entire app — with the purchase stubbed and no
  RevenueCat SDK present, this build should make **no** network calls at all, which makes the
  offline-first claim trivially verifiable now (and gives a clean baseline to compare against
  once billing is added later).

**Week 3 total: ~6h.** **Deliverable:** A fully functional local-first MVP running from a release
build on the emulator, with a working trial gate and paywall UI backed by a stubbed unlock.

---

## 5. Revised Totals

| Phase | Original budget | Revised budget |
|---|---|---|
| Phase 0 (env setup) | — (not budgeted) | ~2.5h |
| Week 1 | 6h | ~8h |
| Week 2 | 6h | ~12h |
| Week 3 | 6h | ~6h |
| **Total** | **18h** | **~28.5h** |

This is roughly **~5 weeks at 6h/week**, not 3. That's the honest cost of the decisions made in
this session (Riverpod over plain Provider, true content-stream redaction over rasterization,
dual review UX instead of one flow, and from-scratch tooling setup) versus the original
overview's 18-hour estimate. Note Week 2 alone is now ~43% of the total — it carries essentially
all the technical risk.

If you want to cut further, the remaining big lever is **rasterization instead of true
content-stream removal**, which would save most of Task 2.4's 4h at the cost of text
quality/searchability on redacted pages. Dropping the manual-review UX would save ~1.5h, but
that's now less attractive: with checksum validation removed, manual review is the primary
false-positive defense.

---

## 6. Open Risks Carried Into Execution

1. **Content-stream text removal may not have a clean public API** in the current Syncfusion
   Flutter PDF package — Task 2.1's spike exists specifically to surface this early, in week 2
   rather than discovering it at Task 2.4.
2. **False positives from shape-only patterns** — with checksums skipped, the IRD and
   generic-numeric patterns will flag unrelated long numbers. Mitigated by the manual-review
   flow, not by the patterns themselves. Consider defaulting the review setting to manual.
3. **Monetization is unvalidated** — the paywall UI and gate are real, but no purchase has ever
   round-tripped through a store. Real billing integration, store product setup, and test-track
   validation all remain unproven until the deferred post-MVP work happens.
4. **Emulator-only testing** means real-device quirks (permission dialogs, storage behavior on
   an actual Android 13+/14+ phone) are unverified until a physical device is available.
