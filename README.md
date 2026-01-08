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
- permissions and manifest indicators
- size / component counts
- other lightweight signals

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
