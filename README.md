# DREAM Android — Professor Demo Brief (Defense-Ready)

This document is meant to handle **cross-questioning**. It explains what happens at:
- **UI level** (browser)
- **API level** (FastAPI endpoints)
- **terminal level** (uvicorn logs + adb/perfetto commands)
- **file/artifact level** (`uploads/`, `traces/`, `csv_out/`, `features/`, `model_out/`)

---

## 1) One-line pitch
DREAM Android is a practical APK/XAPK security analyzer that combines:
- **static inspection** (manifest/permissions/components + lightweight APK metadata)
- **real-device dynamic tracing** (Perfetto on a physical Android device)
- **multi-label ML** (predicts multiple risk labels at once)

to produce **explainable risk labels** and an overall **risk score**.

---

## 2) What the professor should understand first (big picture)
Static analysis can miss runtime-only behaviors:
- delayed execution
- dynamic code loading
- emulator detection

So our system captures runtime evidence on a **real device** and turns it into structured features for ML.

---

## 3) How to run it (terminal commands)
### Backend
From `~/apksec_project`:
```bash
source venv/bin/activate
python -m uvicorn api.main:app --host 0.0.0.0 --port 8000
```

What this “environment” is (important for viva):
- The project runs inside **WSL (Linux)** on your Windows machine.
- `venv` is a **Python virtual environment** that isolates Python packages for this project.
  - This is where FastAPI, scikit-learn, perfetto trace processor Python bindings, pandas, etc. are installed.
- `uvicorn` is the **ASGI server** that runs the FastAPI app (`api.main:app`).
- Dynamic analysis uses **ADB** to communicate with the physical Android device.
  - In this repo we call ADB via `./adb_wrapper.sh` (so WSL can call Windows `adb.exe` reliably).

Expected in terminal:
- `Found model at: model_out/model_labels.joblib`
- `Uvicorn running on http://0.0.0.0:8000`

### Frontend
The UI is a static page (served separately). It talks to FastAPI through HTTP.

---

## 4) Frontend → Backend flow (exact endpoints)
### Device check
The UI periodically calls:
- `GET /api/check_device`

Purpose:
- confirms ADB connectivity
- returns device serial if present

In UI you see:
- Device `Connected` / `Not Connected`

### Analysis request
When you click **Analyze**, the UI does:
- `POST /analyze` with multipart form-data:
  - `file`: APK or XAPK
  - `consent`: (flag to allow dynamic analysis)

While this is running:
- UI shows an overlay “Analyzing APK…”
- UI disables the Analyze button

### Training stats / retrain
The UI polls:
- `GET /admin/stats`

And retrain triggers:
- `POST /retrain`

The stats show:
- total uploads
- trained vs untrained
- last retrain result

---

## 5) What happens inside `/analyze` (backend pipeline)
The backend does (conceptually):

### Step A — store upload
- Writes file under:
  - `uploads/<timestamp>_<original_filename>`
- Inserts upload metadata into `uploads.db`.

### Step B — package name detection (critical)
We must find the **real Android package id** (e.g., `com.facebook.orca`).

Why this matters:
- ADB install/launch/trace naming depend on package id
- If we use a wrong package name, `adb shell monkey -p ...` fails

How we detect:
- **APK**: read `AndroidManifest.xml` (using parsing/fallback tools)
- **XAPK**: parse the XAPK `manifest.json` if present (XAPK is a container of one or more APKs)

In terminal you see:
- `[INFO] Detected package: com.facebook.orca`

### Step C — static analysis
Runs static feature extraction:

In our current implementation, the **exact static feature fields** produced by `extract_static_features(apk_path)` are:
- `apk_size_mb`
- `dex_count`
  - number of `classes*.dex` files inside the APK (proxy for multi-dex / complexity)
- `native_lib_count`
  - number of `lib/*.so` native libraries (native code risk signal)
- `perm_count`
  - count of manifest permissions
- `dangerous_perm_count`
  - a heuristic count of permissions containing keywords like `READ`, `WRITE`, `SEND`, `RECEIVE`, `ACCESS`, `MODIFY`
- `exported_components`
  - a heuristic count based on components with intent-filters (approx proxy for exposure surface)

This is fast (few seconds).

### Step D — dynamic analysis (if device connected + consent)
If a device is available, the backend calls the dynamic runner.
Artifacts produced:
- `traces/<package>.trace`
- `csv_out/<package>.*.csv`
- `features/<package>.features.csv`

Terminal logs typically show:
- `[DYNAMIC] Installing <package>...`
- `[DYNAMIC] Successfully installed <package>`
- `[DYNAMIC] Attempting to launch <package>...`
- `[DYNAMIC] Starting Perfetto trace...`
- `[DYNAMIC] Perfetto trace completed successfully`
- `[OK] Enhanced features written: features/<package>.features.csv`

### Step E — ML inference + risk scoring
If the dynamic features file is found and the model bundle exists:
- backend loads `model_out/model_labels.joblib`
- vectorizes features using the model’s saved feature schema
- predicts probabilities for each risk label

Output is then combined with static baseline to avoid “near-zero” results when static signals already indicate risk.

Return JSON includes:
- overall risk score (0–100)
- high-level risks (% breakdown)
- detailed label probabilities
- flags:
  - `dynamic_used`
  - `dynamic_ok`
  - `ml_effective`
- timing breakdown

---

## 6) Dynamic analysis: what exactly is “monkey fallback”?
### What is `monkey`?
Android Monkey is a tool that sends pseudo-random UI events to an app to simulate interaction.

We use it primarily as a **reliable launcher**:
- Some apps don’t have a single obvious launch activity.
- `am start` may fail depending on exported activities.
- Monkey can launch using the package id.

Conceptually the command is:
```bash
adb shell monkey -p <package> -c android.intent.category.LAUNCHER 1
```

So “monkey fallback” means:
- we tried a direct launch mechanism
- if that fails, we launch via Monkey so the app is actually running during tracing

Why the professor should like this:
- it increases robustness across different apps
- reduces manual intervention

---

## 7) Perfetto: what we capture and why
Perfetto records system-level traces:
- process/thread activity
- scheduling
- CPU and runtime events
- other OS-level signals depending on trace config

Why Perfetto:
- structured, modern tracing
- works on real devices
- more stable than ad-hoc log scraping

Trace duration:
- configured around **~30 seconds** for demo practicality
 - **Where to change it (exact):**
   - API launches dynamic analysis from `api/main.py` with `--duration 30`.
   - The dynamic runner is `tools/run_dynamic_analysis.py` and its `run_dynamic(..., duration: int = 30)` uses `duration_ms: {duration * 1000}` in the generated Perfetto config.
   - So you can change the default in either place, but the API-side `--duration` value will override the script default.

### What our Perfetto config collects (exact)
In `tools/run_dynamic_analysis.py` we generate a minimal perfetto config with these data sources:
- `linux.process_stats`
  - collects process statistics by polling `/proc` periodically (`proc_stats_poll_ms: 1000`)
- `android.surfaceflinger.frametimeline`
  - collects frame timeline events (useful for UI/render related signals)

This trace is then processed offline using `perfetto.trace_processor.TraceProcessor`.

### What tables/CSVs we export from the trace
`extract_csvs.py` attempts to export the following Perfetto SQL tables:
- `android_binder_txns`
- `slice`
- `sched`
- `process`
- `thread`

Each one becomes a CSV file under `csv_out/` named:
- `csv_out/<package>.slice.csv`
- `csv_out/<package>.sched.csv`
- `csv_out/<package>.process.csv`
- `csv_out/<package>.thread.csv`
- `csv_out/<package>.android_binder_txns.csv` (only if present)

Note: depending on device/trace contents, some tables may be missing/empty and are logged as `[SKIP]`.

---

## 8) Files produced (artifacts to show during cross-questions)
After a successful dynamic run for package `com.facebook.orca`, you will see:

- **Upload:**
  - `uploads/<timestamp>_Messenger_...apk`
- **Perfetto trace:**
  - `traces/com.facebook.orca.trace`
- **Extracted CSVs:**
  - `csv_out/com.facebook.orca.process.csv`
  - `csv_out/com.facebook.orca.thread.csv`
  - `csv_out/com.facebook.orca.slice.csv`
- **Final features used by ML:**
  - `features/com.facebook.orca.features.csv`
- **ML model bundle:**
  - `model_out/model_labels.joblib`
- **DB:**
  - `uploads.db`

If professor asks “how do you know dynamic really happened?”:
- point to `features/<package>.features.csv`
- point to uvicorn logs showing install/launch/trace/pull/feature extraction

### What dynamic features are inside `features/<package>.features.csv` (exact columns)
These features are computed by `feature_extractor.py` from the exported trace CSVs:

Basic counts:
- `slice_count`
- `sched_event_count`
- `process_count`
- `thread_count`

Slice duration statistics (derived from `slice.dur`):
- `slice_avg_dur_ms`
- `slice_std_dur_ms`
- `slice_max_dur_ms`
- `slice_p90_dur_ms`
- `slice_cv_dur` (coefficient of variation)
- `slice_max_to_mean`

Normalized ratios:
- `slice_per_thread`
- `sched_per_thread`
- `slice_per_process`
- `threads_per_process`

Binary/threshold indicators:
- `high_background_activity` (1 if `sched_per_thread > 100`)
- `high_thread_pressure` (1 if `threads_per_process > 50`)

---

## 9) ML model details (what it is, and how it handles low data)
### 9.1 Labels predicted (multi-label output)
We do not predict only “malware vs benign”. Instead the model predicts **multiple risk labels simultaneously** (examples from our label set):
- `privacy_leak`
- `tracking_ads`
- `insecure_communication`
- `weak_cryptography`
- `data_exposure`
- `exported_components`
- `native_code_risk`
- `code_obfuscation`
- `malicious_behavior`

Why this matters:
- One app can be safe in one dimension but risky in another.
- The output is easier to explain: “why is it risky?” not just “is it malware?”.

### 9.2 What model is used (exact structure)
The trained file `model_out/model_labels.joblib` is a **model bundle** (saved using `joblib`) containing:
- the trained model object (`PerLabelModel`)
- the **ordered feature schema** (`feature_columns`)
- the list of labels
- a version id (timestamp)

What the “ML model name” is in plain words:
- **Per-label Calibrated Linear SVM bundle** (stored in `model_labels.joblib`).
- Internally: 14 binary estimators (one per label).

The predictor is **per-label binary classification**:
- We train **one binary classifier per label** (14 classifiers).
- At inference time we run all of them and collect probabilities.

Core estimator (for labels with enough data):
- `StandardScaler` + `LinearSVC` (linear SVM)
- wrapped by `CalibratedClassifierCV(method="sigmoid", cv=3)` to produce probabilities.

### 9.3 Why Linear SVM + calibration (latency vs accuracy trade-off)
This choice is specifically for a good **accuracy/latency** balance on CPU:

- **Latency (fast inference):**
  - Linear models compute a dot-product (`w·x + b`) which is very fast.
  - With ~14 labels, we do ~14 linear predictions (still fast).
  - Calibration adds small overhead but remains lightweight compared to tree ensembles.

- **Accuracy (good for sparse/engineered features):**
  - Our feature set is engineered numeric signals (static + dynamic counts/ratios).
  - Linear decision boundaries often work well in this setting.
  - `class_weight="balanced"` helps with imbalanced labels.

- **Why probability calibration:**
  - `LinearSVC` does not natively output probabilities.
  - The UI needs probabilities for risk bars and score composition.
  - Calibration learns a mapping from raw SVM scores to probabilities (sigmoid/Platt scaling), improving interpretability.

In short:
- A deep model would be heavy.
- Random forests / gradient boosting can be slower and harder to calibrate.
- Linear SVM is a strong baseline for speed + reasonable accuracy.

### 9.4 Feature vector (what goes into ML)
At inference time, we build a single feature dict from:
- **Static features:** permissions, manifest/component indicators, size-related signals.
- **Dynamic features:** extracted from Perfetto → CSV → aggregated numeric features.

We then vectorize into a fixed order using the saved `feature_columns` list. This is critical:
- the model expects the same schema at training and inference
- missing features default to 0

### 9.5 Training data and retraining semantics
Training reads a structured dataset (`app_security_dataset.csv`) where:
- each row = one analyzed app
- columns = features + label columns

Retraining combines:
- a baseline dataset (`training_set` / curated examples)
- new labeled uploads recorded in `uploads.db`

#### After each retrain, does `model_out/model_labels.joblib` “add values”?
Not incrementally. Each retrain:
- re-reads the dataset,
- re-fits the estimators,
- and **overwrites** `model_out/model_labels.joblib` with a new bundle.

What changes after retrain:
- the estimator weights/parameters (because training data changed)
- `version` (timestamp)
- potentially `feature_columns` if your dataset schema changed (not recommended to change frequently)

### 9.6 What happens when data is insufficient (robust fallbacks)
Some labels can be rare early on. The training script handles this safely:

- **If a label has only one class present** (all 0 or all 1):
  - uses `DummyClassifier(strategy="constant")`
  - meaning: we do not pretend to learn a boundary when there is no information.

- **If one class is too rare** (minority count < 3):
  - falls back to **Logistic Regression** (still fast, more stable than CV calibration on tiny data)

So the system remains runnable even as the dataset grows over time.

### 9.7 How ML affects the final output
The ML probabilities do not replace static analysis. We combine them:
- static analysis provides a baseline risk probability per label
- ML updates/refines the label probability when dynamic features exist

This avoids the failure mode where “ML outputs near-zero everywhere” even when static signals are clearly risky.

### 9.8 What to say if asked about latency
In the end-to-end pipeline, most time is **dynamic tracing** (~30–60s).
ML inference is typically **milliseconds** relative to trace collection.
So we optimize ML for:
- low inference overhead
- stable probabilities for UI

---

## 10) What the UI shows (and why these fields matter)
After analysis completes, the UI displays:
- **Overall Risk Score** (0–100)
- **Risk tier** (low/medium/high) using current thresholds
- **Confidence**
- **Analysis Mode**
  - indicates whether dynamic was used
- **Dynamic Features: Yes/No**
- **Dynamic OK: Yes/No**
  - shows whether trace + feature extraction succeeded
- **ML Model: Active/Inactive**
- **Package** and **File**
- **Timing**: total/static/dynamic seconds
- **High-level risks** (privacy/security/network/etc.)
- **Detailed security labels** (bars)

If professor asks “how do you know ML was used?”
- show `ML Model: Active`
- show `ML Used: Yes`

---

## 10.1 What exactly is `uploads.db`?
`uploads.db` is a SQLite database used for bookkeeping and retraining UX.

It contains 3 main tables (see `api/storage.py`):

### `uploads`
- `id` (auto increment)
- `filename`
- `package`
- `uploaded_at`
- `consent` (whether dynamic analysis was allowed)
- `used_for_training` (0/1)
- `analysis_json` (cached JSON output for that upload)

### `upload_labels`
- stores per-upload label annotations:
  - `upload_id`, `label`, `value`, `provided_at`, `provided_by`

### `retrain_log`
- stores retrain history:
  - start/end timestamps
  - how many new uploads were used
  - success/failure
  - captured stdout/stderr from training

This DB is what powers the UI line like:
`Uploads: total X, trained Y, untrained Z. Last retrain: ...`

---

## 14) Network capture / PCAPDroid status
PCAPDroid is currently **not integrated in code** (no PCAPDroid/pcap pipeline is executed).
It is mentioned only as an **optional future extension**.

---

## 11) Demo script (defense-ready)
### 0) Pre-demo checklist (30 seconds)
- backend running
- device connected in UI
- model exists at `model_out/model_labels.joblib`

### 1) Start the demo
1. Show UI device status = Connected.
2. Upload an APK (Messenger is a good example).
3. Click Analyze.

### 2) While it runs, narrate what happens
- “We first detect the package id from manifest, because all ADB control depends on package name.”
- “We extract static features immediately.”
- “Then we install and launch on the real device. If direct launch fails, we use Android Monkey as a robust launcher.”
- “We collect a ~30s Perfetto trace, pull it, convert to CSV, and extract structured features.”
- “Then ML predicts multiple risk labels and we combine static + ML outputs.”

### 3) Show evidence
- point to terminal logs and the generated files in `traces/`, `csv_out/`, `features/`.

### 4) Close
- show risk score + top labels
- mention retrain workflow briefly (optional click)

---

## 12) Limitations (answer honestly)
- Real-device requirement limits scalability.
- Monkey interaction is shallow; login-gated features may not be exercised.
- Some malware may delay beyond trace window.
- ML label quality depends on labeling and dataset size.

---

## 13) Future work (concrete)
- UIAutomator scripted interactions for deeper coverage.
- Adaptive trace duration based on early suspicious signals.
- Add stronger network evidence pipeline.
- Better explainability: “why this label fired” with top contributing features.
