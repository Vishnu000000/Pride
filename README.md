# DREAM Android — Professor Demo Brief

## 1) One-line pitch
DREAM Android is a practical APK security analyzer that combines static inspection with real-device dynamic tracing (Perfetto) and a multi-label ML model to produce explainable privacy/security risk labels and an overall risk score.

## 2) What we built (end-to-end workflow)
- **Web UI**: Upload APK/XAPK → Analyze → View risk score + labels → Retrain model.
- **Backend orchestration (FastAPI)**:
  - Stores uploads + metadata in **SQLite (`uploads.db`)**
  - Runs static feature extraction from APK
  - Optionally runs dynamic analysis on a **real Android device** via **ADB**
  - Runs ML inference using a trained model bundle
  - Returns a single JSON result with:
    - overall risk score
    - high-level risk breakdown
    - detailed labels
    - whether dynamic + ML were actually used
    - timing (static/dynamic/total)

## 3) Dynamic analysis (what happens on the device)
- Installs the app on the connected device.
- Launches the app (monkey fallback) to trigger runtime behavior.
- Records a **Perfetto trace (~30s)**.
- Pulls trace back to host and extracts CSV signals.
- Extracts dynamic features into a fixed schema feature row.

## 4) ML model (how it is used)
- We train a **multi-label** classifier (predicts multiple risk labels at once).
- Training data sources:
  - curated dataset (`training_set`)
  - labeled uploads from the UI workflow
- Prediction uses **static + dynamic** features (when dynamic is available).
- Output is a set of **per-label probabilities**, which are combined with a static baseline (to avoid near-zero results on real apps).

## 5) What we tackled / major engineering problems solved
- **Real package name detection (APK + XAPK)**
  - Robustly extracts `com.*` identifiers so install/launch works reliably.
- **Reliable dynamic pipeline**
  - Trace capture + pull + CSV extraction + feature extraction are now consistent.
- **Dynamic feature naming consistency**
  - Ensures dynamic features are non-zero and ML actually gets used.
- **Believable risk score calibration**
  - Risk score remains monotonic with model outputs while producing demo-realistic values.
- **Retrain UX**
  - Shows trained/untrained counts and last retrain status.
- **Device workflow stability**
  - Recheck device status and run dynamic only when device is available.

## 6) Demo script (2–3 minutes)
1. **Show device connected** in UI.
2. Upload an APK (example: Messenger) → click **Analyze**.
3. Explain the pipeline while it runs:
   - static signals extracted
   - app installed/launched
   - Perfetto trace captured
   - features extracted
   - ML predicts multiple labels
4. Show results:
   - overall risk score + tier
   - high-level risks
   - detailed labels
   - dynamic OK + ML active
   - timing breakdown
5. (Optional) click **Retrain Model** and show stats update.

## 7) Current system status (from latest run)
- Device connected and dynamic traces succeed.
- Example run:
  - Package: `com.facebook.orca`
  - Dynamic OK: Yes
  - ML Active: Yes
  - Total time: ~56.7s (static ~3.8s, dynamic ~52.9s)

## 8) Limitations (honest)
- **Scalability**: relies on real devices; parallelism is limited.
- **Interaction depth**: monkey automation may not reach login-gated behavior.
- **Delayed malware**: sophisticated malware can sleep/trigger later than the trace window.
- **Labeling quality**: ML quality depends heavily on labeling correctness and dataset diversity.

## 9) Future work (practical next steps)
- Better interaction automation (UIAutomator scripts / recorded flows).
- Longer or adaptive tracing windows for suspicious apps.
- Richer network evidence (optional PCAPDroid, DNS/HTTP metadata features).
- Stronger model evaluation report (time-based split + ablations).
- Explainability improvements (top features per label, evidence snippets).
