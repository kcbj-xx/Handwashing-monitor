# AI Handwashing Tracker Blueprint & Specification

## 1. Project Overview
- **Core Purpose:** A zero-clutter, installable/downloadable Progressive Web App (PWA) supporting both **portrait and landscape** modes on phones, tablets, and wash-station kiosks, guiding workers through the 6 core WHO handwashing motions (18 seconds total, 3s per step) using real-time front-camera AI hand contact and velocity tracking to gate timer progression.
- **Target User:** Food processing workers, commercial kitchen staff, and quality control (QC) hygiene supervisors operating at wash stations.

## 2. Tech Stack & Constraints
- **Frontend:** HTML5 (`index.html`), Tailwind CSS, Custom 2D Anatomical SVG Icon Library, Service Worker (`sw.js`), Web App Manifest (`manifest.json` with `"orientation": "any"` and `192x192` / `512x512` PNG icons in `icons/`).
- **AI Vision Engine:** `@mediapipe/hands`, `@mediapipe/camera_utils`, and `@mediapipe/drawing_utils` running locally in the browser via WebGL/WASM at 30 FPS.
- **Backend/Database:** Client-side local execution (zero video leaves the device) with `localStorage` and Custom DOM Events (`handwash:complete`) for QC compliance timestamp logging.
- **State Management:** Reactive `requestAnimationFrame` state machine (`START` → `WASHING` → `COMPLETE`) synchronized with millisecond-precision DOM updates and a `300ms` micro-pause debounce buffer.
- **Strict Rules:**
  - Responsive across **both portrait and landscape modes** and small phone displays with safe-area inset support and installable PWA support (`beforeinstallprompt` + iOS/Android install guide).
  - Zero-clutter Main Wash Interface limited strictly to **5 core UI elements** (Top-Left Step Timer, Top-Right Camera PiP, Center Animated WHO Hand Prompt + Label, Bottom 6-Segment "Just Dance" Progress Bar, Bottom-Left `↺` Reset Button).
  - Timer and progress bar advance **only** when the worker's two hands match the **active step's specific 2D landmark pose geometry** AND frame-to-frame Euclidean + inter-hand relative scrubbing velocity exceeds the step threshold (`isPoseValid && velocity >= velocityThreshold`), freezing at the exact millisecond when motion pauses or the gesture pose is broken.
  - **50/50 Opposite-Hand Split (Steps 2–6):** All steps except Step 1 split their duration equally between the initial hand configuration (first `1.5s`) and the **opposite hand** (remaining `1.5s`), automatically mirroring the central 2D hand illustration, updating all 5 language prompts (`English`, `Chinese`, `Spanish`, `Indian/Hindi`, `Vietnamese`), and requiring the worker to switch hands at the 50% midpoint.
  - **Extended Completion Screen + Next User Button:** Holds the `COMPLETE` screen for `15000ms` (15 seconds) before auto-closing, and includes a high-contrast circular **`→`** arrow button so the next worker in line can immediately advance to their turn.

## 3. Core Architecture & Data Models
- **WHO Step Schema (`WHO_STEPS`):**
  ```json
  {
    "id": "number (1–6)",
    "whoStep": "string (Step 2 – Step 7)",
    "label": "string (Primary English action caption)",
    "translations": {
      "zh": "string (1. Chinese / 中文)",
      "es": "string (2. Spanish / Español)",
      "hi": "string (3. Indian - Hindi / हिन्दी)",
      "vi": "string (4. Vietnamese / Tiếng Việt)"
    },
    "duration": "number (3.0 seconds per step; 18.0 seconds total)",
    "svgType": "'palm-to-palm' | 'interlaced' | 'dorsum' | 'knuckles' | 'thumbs' | 'fingertips'",
    "velocityThreshold": "number (Normalized Euclidean landmark displacement threshold)"
  }
  ```
- **Motion Engine State Schema:**
  ```json
  {
    "appState": "'START' | 'WASHING' | 'COMPLETE'",
    "currentStepIndex": "number (0–5)",
    "stepRemainingMs": "number (Millisecond countdown for active gesture)",
    "isScrubbingActive": "boolean (RUNNING vs PAUSED after 300ms debounce)",
    "handsInFrameCount": "number (0–2 detected hands)",
    "isPoseValid": "boolean (Two-hand proximity & wash-zone validation)",
    "currentVelocity": "number (Mean Euclidean displacement across wrist, MCP, and fingertip landmarks)"
  }
  ```
- **QC Compliance Log Schema (`handwash_qc_logs`):**
  ```json
  {
    "completedAt": "string (ISO-8601 timestamp)",
    "totalDurationSec": "number (18)",
    "stepsCompleted": "number (6)"
  }
  ```

## 4. Feature Roadmap (Scope)
- [x] **Milestone 1: UI Shell & Anatomical SVG Hand Assets**
  - 3-screen landscape layout (`START`, `WASHING`, `COMPLETE`).
  - Custom 2D anatomical hand SVG library for WHO Steps 2–7 with directional motion arrows and mini timeline icons.
  - 6-block segmented bottom progress bar ("Just Dance" style) and bottom-left circular `↺` Reset button.
- [x] **Milestone 2: Motion-Gated AI Engine**
  - Mirrored front-camera PiP feed with real-time MediaPipe skeleton overlay.
  - Frame-to-frame Euclidean velocity calculation $\Delta d = \sqrt{(x_t - x_{t-1})^2 + (y_t - y_{t-1})^2}$ across wrist (`0`), MCP knuckle (`5, 9, 13, 17`), and fingertip (`4, 8, 12`) landmarks.
  - Millisecond-accurate freeze/resume timer loop with a `300ms` micro-pause smoothing buffer, 50/50 opposite-hand midpoint switch on Steps 2–6, and a `15000ms` completion screen with a circular `→` arrow button.
- [x] **Milestone 3: Wash-Station Floor Calibration**
  - Interactive **Gesture Calibration** button (`Calibrate Gestures`) anchored in the lower-right corner of the Start Menu (`#start-screen`, which balances 2 languages above and 2 languages below `START`).
  - Live camera & skeleton calibration panel with per-step testing across all 6 WHO gestures, a 3-second `Auto-Sample (3s)` calibration routine, Pose Tolerance (`0.75x–1.45x`) & Velocity Threshold (`0.50x–1.50x`) sliders, and Opposite-Hand Switch toggle persisted in `localStorage`.
  - **Touching-Hand Merged Skeleton Support & Palm vs. Back (`DORSUM`) Differentiation:**
    - When two hands touch closely, MediaPipe's Non-Maximum Suppression frequently outputs a single merged skeleton (`hands.length === 1`). The engine handles both `2` skeletons and `1` touching skeleton (verified via adaptive skin/glove pixel mass around the palm in `detectTouchingHandMass`).
    - Computes rotation-invariant 2D cross-product palm winding $(\text{MiddleMCP}_9 - \text{Wrist}_0) \times (\text{IndexMCP}_5 - \text{PinkyMCP}_{17})$ combined with MediaPipe handedness in `computeHandSurfaceOrientation` to distinguish **PALM (Front)**, **BACK (Dorsum)**, and **EDGE**, ensuring Palm-to-Palm (Step 1) and Back-to-Palm (Step 2) are strictly differentiated.

## 5. UI/UX & Design System
- **Theme:** High-contrast industrial dark mode (`#090d16` background) engineered for instant legibility at arm's length in bright or humid food-processing wash stations.
- **Typography:** `Inter` (`font-extrabold` / `font-black`), oversized tabular numerals for countdown readability, and uppercase tracking on primary prompts.
- **Primary Colors:**
  - **Canvas Background:** Deep Obsidian Slate (`#090d16` / `slate-900`)
  - **Active / Valid Scrubbing (`RUNNING`):** Emerald Green (`#10b981` / `#34d399`) with glowing border indicator
  - **Paused / Stationary Hands (`PAUSED`):** Amber Warning (`#f59e0b` / `#fbbf24`) with pulsing central hand prompt
  - **Hands Out of Frame (`PAUSED`):** Rose Alert (`#f43f5e`) on the Camera PiP border
  - **Neutral / Upcoming Elements:** Cool Slate (`#94a3b8` / `#64748b`) and crisp White (`#ffffff`) directional motion arrows
