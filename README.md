# DREAM Android: Dynamic Runtime Evaluation Analysis of Malware for Android (Updated Poster Text Draft)

## Title
**DREAM Android: Dynamic Runtime Evaluation Analysis of Malware for Android**

## Authors
Kilaparthi Vishnu Vardhan
Prof. Chester Rebeiro
Department of Computer Science and Engineering, IIT Madras

## Introduction & Motivation
Android apps routinely handle sensitive data (contacts, location, payments, authentication tokens). However, **what is visible in an APK is not always what happens at runtime**. Malicious or risky apps may:
- hide behavior behind dynamic loading/obfuscation,
- delay actions,
- change behavior depending on environment.

Therefore, **static scanning alone is often insufficient**.

We present DREAM Android, a practical system that runs apps on a **real Android device**, captures runtime evidence, and produces **explainable privacy/security risk scores and labels**.

## Problem Definition
**Goal:** Automatically evaluate Android apps using static + runtime evidence and produce meaningful, explainable risk outputs.

**Challenges:**
- Emulators are detectable by advanced malware.
- Raw traces/logs are large and hard to compare across apps.
- Manual review does not scale.
- Results must be interpretable for non-experts.

## System Overview (Architecture)
1. User uploads an **APK/XAPK** via a lightweight web UI.
2. Backend performs **static feature extraction** (manifest + permissions + app metadata).
3. If a device is connected, backend performs **dynamic analysis**:
   - installs the app,
   - launches it (monkey fallback),
   - captures a **Perfetto trace (~30s)**,
   - extracts CSV signals and dynamic features.
4. A **multi-label ML model** predicts multiple risk categories.
5. UI renders:
   - overall risk score,
   - high-level risk breakdown,
   - detailed label probabilities,
   - whether dynamic/ML were actually used,
   - timing for static/dynamic/total.

## Key Ideas
- Runs apps on a **real Android device** (not an emulator), increasing chances of observing hidden behavior.
- Uses **Perfetto** for robust system-level tracing.
- Combines **static + dynamic features**.
- Multi-label classification: outputs multiple risks simultaneously (privacy, security, network, behavior, etc.).
- Supports **continuous retraining** as new labeled apps are added.

## Tools & Technologies
- **ADB**: install/launch apps and collect artifacts.
- **Perfetto + Trace Processor**: runtime traces and structured CSV extraction.
- **FastAPI**: analysis orchestration + UI endpoints.
- **Pandas/NumPy**: feature cleaning/aggregation.
- **Scikit-Learn**: ML training and inference.
- **SQLite (`uploads.db`)**: upload metadata + retraining bookkeeping.
- **PCAPDroid (optional)**: network capture for future expansion.

## Dataset & Deployment Setup (Current)
- Input formats: **APK + XAPK** supported.
- Static features: always available.
- Dynamic features: available when tracing completes successfully on device.
- Retraining workflow: uses **training_set + labeled uploads**, tracked via backend stats.

## Machine Learning Workflow
- **Structured dataset formation**: static + dynamic traces are normalized into a fixed-schema feature vector.
- **Multi-label classification**: predicts multiple risk labels per app.
- **Retraining**: controlled retraining when new labeled uploads are added.

## Results (Update with latest numbers before printing)
- Device-based dynamic tracing is stable and produces features reliably.
- Example application run (Messenger):
  - package: `com.facebook.orca`
  - dynamic OK: Yes
  - ML active: Yes
  - overall risk score: ~34% (UI tiering configurable)
  - total time: ~57s (static ~4s, dynamic ~53s)

## Limitations and Future Work
**Limitations:**
- Real-device requirement limits scalability.
- Monkey-driven interaction may not reach login-gated behaviors.
- Delayed malware may trigger after the trace window.
- Model performance depends on labeling quality and dataset diversity.

**Future Work:**
- Smarter UI automation (UIAutomator recorded flows).
- Longer/adaptive tracing windows.
- Richer network + file I/O features.
- Explainable ML (feature-attribution per label) and stronger evaluation.

## Contact / Reference
IIT Madras, CSE
(Insert your official email IDs here)
