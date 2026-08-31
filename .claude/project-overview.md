# Project Overview: Local-First PDF Redactor (Flutter)

Flutter is the chosen framework for this offline-first utility — it offers high-performance
document rendering, strict cross-platform consistency across iOS and Android, and a powerful
ecosystem for local file operations. All processing happens on-device; no network calls are
made during document selection, redaction, or saving.

## Core Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Mobile App UI                        │
│                Flutter / Dart (BLoC / Provider)          │
└───────────┬─────────────────────────────────┬───────────┘
            │                                 │
            ▼                                 ▼
┌─────────────────────────┐       ┌───────────────────────┐
│     PDF Engine          │       │     PII Engine        │
│    syncfusion_flutter   │       │   Dart Regex /        │
│   (Local Render & Edit) │       │   Lightweight NLP     │
└─────────────────────────┘       └───────────────────────┘
```

## Key Tech Stack Components

### 1. Cross-Platform Framework
- **Choice:** Flutter + Dart
- **Why:** Delivers native compile performance and identical UI rendering across iOS and
  Android. Dart's strong typing and Ahead-Of-Time (AOT) compilation make file processing fast
  and memory-efficient on mobile devices.

### 2. Local PDF Parsing, Rendering & Editing
- **Document Viewer & Editing:** `syncfusion_flutter_pdfviewer` & `syncfusion_flutter_pdf`
- **Why Syncfusion:** Provides robust, native-speed PDF rendering alongside a complete
  low-level PDF manipulation API. Supports searching text positions within pages, overlaying
  blackout shapes, and removing the underlying string nodes/text content directly from the
  document's page content stream before saving.
- **Alternative Lightweight Library:** `pdf` (Dart package for creation/manipulation) +
  `flutter_pdfview` for simple page rendering.

### 3. On-Device PII Detection & Text Extraction
- **Pattern Matching:** Standard Dart Regex (`RegExp`) tailored for regional identifiers
  (tax numbers, passports, SSNs, phone numbers, email patterns).
- **Text Extraction & Processing:** `syncfusion_flutter_pdf`'s `PdfTextExtractor` reads all
  page text locally.
- **On-Device OCR (for scanned PDFs):** `google_mlkit_text_recognition` (runs 100% on-device
  for both iOS and Android).

### 4. Local Storage & File System
- **File Access:** `path_provider` + standard Dart `dart:io` `File` operations to read and
  write PDF binaries without network calls.
- **App Preferences:** `shared_preferences` or `hive` (ultra-fast, lightweight key-value
  database written in pure Dart) for saving custom redaction rules, recent history, and
  templates.

### 5. In-App Purchase & Monetization
- **Monetization SDK:** RevenueCat (`purchases_flutter`) or `in_app_purchase` (official
  Flutter plugin).
- **Why RevenueCat:** Simplifies single-time purchase verification, handles receipt
  validation with App Store and Google Play, and simplifies paywall management.

## Development Setup & Dev Tools
- **IDE:** VS Code or Android Studio with Flutter/Dart extensions.
- **State Management:** `flutter_bloc` or `provider` (keeps document processing logic cleanly
  separated from UI rendering).
- **Build / CI Pipeline:** Codex or Flutter CLI local builds (`flutter build ipa` /
  `flutter build appbundle`).

## MVP Roadmap — 3 Weeks × 6 Hours (18 Total Hours)

```
Week 1: Document Engine & PDF Viewing        [▓▓▓▓▓▓░░░░░░░░░░░░] 6h
Week 2: PII Detection & Local Redaction      [▓▓▓▓▓▓▓▓▓▓▓▓░░░░░░] 6h
Week 3: Storage, UI Polish & Paywall         [▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓] 6h
```

### Week 1: Project Setup & Local PDF Rendering
**Goal:** Open a local PDF file and render its pages smoothly. **Budget:** 6 hours.

- **Task 1.1 — Project Scaffolding & Dependency Setup (1.5 hrs)**
  Initialize a clean Flutter project (`flutter create pdf_redactor`). Add core dependencies
  to `pubspec.yaml`: `syncfusion_flutter_pdfviewer` & `syncfusion_flutter_pdf` (or `pdf`
  package), `file_picker` (for selecting local PDF files), `path_provider` (for local file
  system operations), `provider` or `flutter_bloc` (for state management).
- **Task 1.2 — File Picker & Document Loader (2 hrs)**
  Build a simple home screen with an "Open Document" CTA. Implement `file_picker` to let
  users pick a local `.pdf` file from device storage. Handle local file permission requests
  cleanly.
- **Task 1.3 — PDF Viewer Screen (2.5 hrs)**
  Create a viewer screen using `SfPdfViewer.file()`. Verify page navigation, zoom gestures,
  and multi-page document rendering.

**Deliverable:** A working mobile screen where a user can select any local PDF file from
their device and read through its pages.

### Week 2: PII Text Extraction & Local Redaction Engine
**Goal:** Detect sensitive text patterns and permanently strip them from the PDF. **Budget:**
6 hours.

- **Task 2.1 — Regex Pattern Engine (1.5 hrs)**
  Implement a utility class with standard regex patterns for common PII: email addresses,
  phone numbers, tax/ID numbers (e.g., SSN, NZ IRD, generic numeric identifiers), and custom
  keyword search input.
- **Task 2.2 — Text Extraction & Search Match Positioning (2 hrs)**
  Use `PdfTextExtractor` to scan pages and return match bounds (rectangle coordinates) for
  matched strings. Highlight matches on the UI layer over the PDF viewer so the user can
  review before redacting.
- **Task 2.3 — Redaction Execution & Export (2.5 hrs)**
  Build the redaction method using `syncfusion_flutter_pdf`: draw opaque black rectangles
  over the matched coordinates, clear/remove the underlying text string nodes from the page
  content stream to ensure text cannot be selected or extracted, and save the modified PDF
  binary back to local device storage using `path_provider`.

**Deliverable:** A functional pipeline where searching a keyword (or regex) highlights
matches on screen, and tapping "Redact & Save" outputs a sanitized PDF file with blacked-out
text.

### Week 3: In-App Purchase Gate, Polish & Export Workflow
**Goal:** Add the 3-page free trial limit, single-time purchase paywall, and package for
distribution. **Budget:** 6 hours.

- **Task 3.1 — Free Trial Gate & Page Count Limit (1.5 hrs)**
  Implement logic to count the pages of loaded documents. Allow free processing for documents
  up to 3 pages. Block full redaction export for documents over 3 pages without unlocking.
- **Task 3.2 — RevenueCat Integration (2 hrs)**
  Add `purchases_flutter` dependency and configure a RevenueCat app project. Set up a single
  non-consumable product ($4.99 USD one-time purchase). Build a simple, clean paywall bottom
  sheet modal triggered when limits are reached or when tapping "Unlock Full Version".
- **Task 3.3 — UI Polish, Dark Mode & Local Share (1.5 hrs)**
  Add a "Share Sanitized PDF" button using `share_plus` to send processed PDFs directly via
  Email, Messages, or Files. Light UI polish (clean button styles, loading indicators during
  PDF processing).
- **Task 3.4 — Build Verification & Test Run (1 hr)**
  Run local production builds (`flutter build appbundle` / `flutter build ipa`) on local
  simulators and test devices. Confirm zero network traffic during the entire document
  selection, redaction, and saving workflow.

**Deliverable:** A fully functional MVP ready for beta testing via Apple TestFlight and
Google Play Internal Testing.
