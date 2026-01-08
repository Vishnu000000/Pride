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

## 10) Terminal-level: what actually happens (commands + artifacts)

### Backend start
- Start backend:
  - `python -m uvicorn api.main:app --host 0.0.0.0 --port 8000`
- Expected log:
  - `Found model at: model_out/model_labels.joblib`

### Analyze request lifecycle (high level)
When the UI calls `POST /analyze`, the backend:
- Saves upload into `uploads/` (timestamped filename)
- Detects the Android package name (APK or XAPK)
- Runs static feature extraction
- Runs dynamic analysis (if enabled + device is connected)
- Loads latest dynamic features from `features/`
- Runs ML inference and returns a JSON response

### Dynamic analysis: device + perfetto + processing
The dynamic runner (`tools/run_dynamic_analysis.py`) performs:
- Install:
  - `adb install -r <apk>` (or `adb install-multiple -r ...` for split APKs)
- Launch:
  - tries `adb shell am start -n <pkg>/<activity>`
  - fallback: `adb shell monkey -p <package> -c android.intent.category.LAUNCHER 1`
- Trace capture (~30s):
  - `adb shell perfetto --txt --config - --out /data/misc/perfetto-traces/<package>.trace`
- Pull trace:
  - `adb pull /data/misc/perfetto-traces/<package>.trace traces/<package>.trace`
- Extract CSVs:
  - `python extract_csvs.py traces/<package>.trace csv_out/`
- Extract features:
  - `python feature_extractor.py <package>`

### Files to point to during cross-questioning
- **Uploaded APKs**: `uploads/<timestamp>_<filename>.apk`
- **Perfetto traces**: `traces/<package>.trace`
- **Extracted CSV evidence**: `csv_out/<package>.*.csv`
- **Final dynamic feature row**: `features/<package>.features.csv`
- **Trained model bundle**: `model_out/model_labels.joblib`
- **Upload metadata DB**: `uploads.db`

### Recognizing success/failure quickly in logs
- Dynamic success indicators:
  - `Successfully installed <package>`
  - `Successfully launched <package> using monkey`
  - `Perfetto trace completed successfully`
  - `Enhanced features written: .../features/<package>.features.csv`
- Common non-fatal noise:
  - PTY warning about stdin not being a terminal (trace still succeeds)

## 11) Frontend-level: what the UI calls and how it maps to results

### API calls made by the UI
- **Device status**:
  - `GET /api/check_device`
  - Used to show: Connected / Not connected + device id
- **Train stats**:
  - `GET /admin/stats`
  - Used to show: total uploads, trained/untrained counts, last retrain summary
- **Analyze**:
  - `POST /analyze` with multipart form data `file=<apk/xapk>`
- **Retrain**:
  - `POST /retrain`

### Key response fields used by UI (what to explain)
- `risk_score`:
  - overall risk percent shown in big text
- `analysis_level`:
  - `with_dynamic` vs `static_only`
- `meta.dynamic_used`:
  - whether dynamic features were found and used
- `meta.dynamic_ok`:
  - whether the dynamic pipeline succeeded
- `meta.ml_effective`:
  - whether ML was applied (model loaded + features available + predict succeeded)
- `protocol_percentages`:
  - per-label probabilities shown as “Detailed Security Labels”
- `high_level_percentages`:
  - grouped risk categories shown in “High-Level Risks”
- `timing.total_s/static_s/dynamic_s`:
  - used for the time breakdown line

### Where the logic lives (backend entry points)
- `POST /analyze` endpoint in `api/main.py`
- Core scoring happens in `analyze_apk_core(...)`
- Dynamic orchestration shells out to `tools/run_dynamic_analysis.py`
- Model training is `train_labels.py` (saved to `model_out/model_labels.joblib`)

## 12) Cross-questioning (expected questions + short answers)

### Q: Why not only static analysis?
A: Static can miss runtime-only behaviors (delayed actions, dynamic loading, environment-triggered behavior). Dynamic tracing on a real device captures what actually happens.

### Q: Why real device instead of emulator?
A: Many malware families detect emulators and hide behavior. Real-device execution reduces that evasive gap.

### Q: What exactly do you capture in dynamic analysis?
A: A Perfetto trace that can yield process/thread scheduling signals and app activity summaries; then we convert to CSVs and finally to a fixed feature table so apps are comparable.

### Q: How do you know ML is actually used?
A: The response contains `meta.ml_effective=true` and the UI shows “ML Model: Active”. This only happens when a model is loaded and dynamic features are available.

### Q: Can dynamic fail? What happens then?
A: Yes (ABI mismatch, install/launch failure, tracing failure). In that case we still return static analysis and mark dynamic/ML as inactive so the UI is honest.

### Q: What is your model and why this choice?
A: Multi-label, per-label classifiers with calibrated probabilities. SVM is fast at inference and calibration provides probabilities for UI risk scoring.

### Q: Is the model trained on enough data?
A: Some labels may be rare in the dataset; the training script falls back to simpler classifiers for rare labels to avoid instability. As we collect more labeled apps, per-label performance improves.

### Q: What makes the score “explainable”?
A: We show per-risk-label probabilities and grouped high-level categories, rather than only a single “malicious/benign” bit.

### Q: What are the main limitations?
A: Scalability (real devices), limited interaction depth (monkey), and delayed malware may trigger outside the trace window.
