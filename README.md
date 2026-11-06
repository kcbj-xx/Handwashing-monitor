# AI Handwashing Tracker Blueprint & Specification

## 1. Project Overview
- **Core Purpose:** A zero-clutter, landscape-locked Progressive Web App (PWA) that guides workers through the 6 core WHO handwashing motions (48 seconds total) using real-time front-camera AI hand pose and velocity tracking to gate timer progression.
- **Target User:** Food processing workers, commercial kitchen staff, and quality control (QC) hygiene supervisors operating at wash stations.

## 2. Tech Stack & Constraints
- **Frontend:** HTML5, Tailwind CSS, Custom 2D Anatomical SVG Icon Library, Web App Manifest (`manifest.json` with `"orientation": "landscape"`).
- **AI Vision Engine:** `@mediapipe/hands`, `@mediapipe/camera_utils`, and `@mediapipe/drawing_utils` running locally in the browser via WebGL/WASM at 30 FPS.
- **Backend/Database:** Client-side local execution (zero video leaves the device) with `localStorage` and Custom DOM Events (`handwash:complete`) for QC compliance timestamp logging.
- **State Management:** Reactive `requestAnimationFrame` state machine (`START` → `WASHING` → `COMPLETE`) synchronized with millisecond-precision DOM updates and a `300ms` micro-pause debounce buffer.
- **Strict Rules:**
  - Locked strictly to **landscape mode** via Web App Manifest, Screen Orientation API, and CSS `@media (orientation: portrait)` rotation enforcement.
  - Zero-clutter Main Wash Interface limited strictly to **5 core UI elements** (Top-Left Step Timer, Top-Right Camera PiP, Center Animated WHO Hand Prompt + Label, Bottom 6-Segment "Just Dance" Progress Bar, Bottom-Left `↺` Reset Button).
  - Timer and progress bar advance **only** when valid two-hand pose and frame-to-frame Euclidean scrubbing velocity exceed the step threshold (`isPoseValid && velocity >= velocityThreshold`), freezing at the exact millisecond when motion pauses.

## 3. Core Architecture & Data Models
- **WHO Step Schema (`WHO_STEPS`):**
  ```json
  {
    "id": "number (1–6)",
    "whoStep": "string (Step 2 – Step 7)",
    "label": "string (Concise action caption displayed beneath hand icon)",
    "duration": "number (8.0 seconds per step; 48.0 seconds total)",
    "svgType": "'palm-to-palm' | 'dorsum' | 'interlaced' | 'knuckles' | 'thumbs' | 'fingertips'",
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
    "totalDurationSec": "number (48)",
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
  - Millisecond-accurate freeze/resume timer loop with a `300ms` micro-pause smoothing buffer and `3000ms` auto-reset completion screen.
- [x] **Milestone 3: Wash-Station Floor Calibration**
  - Tuned detection/tracking confidence (`0.45`) and step-specific velocity thresholds for wet-skin specular glare.
  - Two-hand occlusion grace window (`650ms`) to prevent false pauses when palms tightly eclipse each other during scrubbing.

## 5. UI/UX & Design System
- **Theme:** High-contrast industrial dark mode (`#090d16` background) engineered for instant legibility at arm's length in bright or humid food-processing wash stations.
- **Typography:** `Inter` (`font-extrabold` / `font-black`), oversized tabular numerals for countdown readability, and uppercase tracking on primary prompts.
- **Primary Colors:**
  - **Canvas Background:** Deep Obsidian Slate (`#090d16` / `slate-900`)
  - **Active / Valid Scrubbing (`RUNNING`):** Emerald Green (`#10b981` / `#34d399`) with glowing border indicator
  - **Paused / Stationary Hands (`PAUSED`):** Amber Warning (`#f59e0b` / `#fbbf24`) with pulsing central hand prompt
  - **Hands Out of Frame (`PAUSED`):** Rose Alert (`#f43f5e`) on the Camera PiP border
  - **Neutral / Upcoming Elements:** Cool Slate (`#94a3b8` / `#64748b`) and crisp White (`#ffffff`) directional motion arrows
