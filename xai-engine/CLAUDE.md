# xai-engine — Lipika's scope (Phases 4–6)

The AI/intelligence module: turn the mobile client's raw sensor + audio stream into a verified,
explainable emergency decision. See `../CLAUDE.md` for the full project + pipeline.

Your three phases:
- **Phase 4 — Sensor fusion & feature extraction:** clean the raw signals, synchronize timestamps,
  compute a compact feature vector (e.g. `PeakAcceleration`, `MotionVariance`, `AudioEnergy`,
  `GPSVelocity`).
- **Phase 5 — On-device inference:** run an optimized **TensorFlow Lite** model for sub-second
  Emergency/Not-Emergency classification → `{label, confidence}`.
- **Phase 6 — Decision Engine:** compare confidence to a threshold. If it clears, proceed to
  Phase 7 (Gagan); if not, discard and return to Phase 0 silently (target false-positive rate
  < 5%).
- Server-side **SHAP/LIME** explanation logs run **asynchronously in the backend** (Phase 11,
  Chirag's `c2-backend`), not on the phone — coordinate the feature-vector format with him so his
  explainer sees the same features.

## Critical architectural point: inference runs ON-DEVICE

Per the design, Phase 5 is on-device (TFLite in the Android app), so the split is:
- **You train in Python** (scikit-learn / TF), then export a **`.tflite`** model.
- **Feature extraction must run on-device in Kotlin** so the live features match training exactly.
  So your deliverable is either the Kotlin feature-extraction code, or a precise spec Aarush can
  implement — the feature list, order, units, and the time window each is computed over.
- The `.tflite` model + feature spec + decision threshold get integrated into `mobile-client`
  (it will need the TFLite Android dependency added).

Keep a single source of truth for the feature vector (names + order + units + window) shared
between your Python training and the on-device Kotlin — a mismatch there silently wrecks accuracy.

## Your input: the `SensorPacket` contract (Phase 3 → 4)

Produced by the mobile client during an active Ghost State session. **Producer:**
`mobile-client/app/src/main/java/com/incog/mobileclient/ghost/GhostStateService.kt` →
`buildSensorPacket()`, currently emitted every 2s (the `logSnapshot()` seam — your consumer plugs
in there). **Definition:**
`mobile-client/app/src/main/java/com/incog/mobileclient/handoff/SensorPacket.kt`.

```
SensorPacket(
    sessionId:        String            // "SESS-XXXXXXXX", one per activation
    timestampMs:      Long              // wall-clock at packet build
    latestAccel:      Vec3Reading?      // most recent accelerometer sample
    latestGyro:       Vec3Reading?      // most recent gyroscope sample
    latestLocation:   LocationReading?  // most recent GPS fix (null until first fix)
    accelSamples:     List<Vec3Reading> // rolling history, ≤1000 samples
    gyroSamples:      List<Vec3Reading> // rolling history, ≤1000 samples
    audioRmsEnergy:   Double            // rolling RMS of the mic buffer
    audioBufferedMs:  Long              // ms of audio held in the 30s ring buffer
)

Vec3Reading(timestampMs: Long, x: Float, y: Float, z: Float)
    // accelerometer x/y/z in m/s² (includes gravity); gyroscope x/y/z in rad/s
LocationReading(timestampMs: Long, latitude: Double, longitude: Double,
                speedMps: Float, accuracyM: Float)
```

**Rates & formats:**
- Accel + gyro sampled at `SENSOR_DELAY_GAME` (~50 Hz, ~20 ms). History caps at 1000 samples
  (~20 s) then rolls.
- Audio: 16 kHz, mono, PCM 16-bit, held in a 30-second in-memory circular buffer.

## Open coordination items with Aarush (mobile-client)

1. **Raw audio access.** `SensorPacket` currently exposes only `audioRmsEnergy` +
   `audioBufferedMs`, NOT the raw PCM. If your acoustic model (e.g. RAVDESS distress detection)
   needs raw samples/MFCCs, the mobile app must expose the ring buffer — ask Aarush to add an
   accessor on `AudioBufferCollector` (the buffer already exists in memory,
   `sensors/AudioBufferCollector.kt`). Agree on the exchange format (raw PCM vs pre-computed
   features on-device).
2. **Delivery cadence.** Packets are produced every 2 s right now; if your model wants a different
   window/stride, say so.
3. **Stopping a session.** Your Phase 6 "not an emergency" path should end Ghost State — the app
   already has a stop entry point (`GhostStateService.stop(context)` / the stand-down flow). Agree
   on how your decision calls it.

## What you deliver

- A trained **`.tflite`** emergency-classification model.
- The **feature-vector spec** (names, order, units, window) — the shared contract for Python
  training and on-device Kotlin extraction.
- The **decision threshold** + Phase 6 logic.
- Coordinated with Chirag: the same feature format for the server-side SHAP/LIME explainer.

## Datasets (from the project report)

- **UCI HAR** — accelerometer/gyroscope activity recognition (motion features).
- **WISDM** — smartphone activity motion data.
- **RAVDESS** — emotional speech / acoustic distress markers.
- **SisFall** — high-force fall / kinetic-impact detection.
(StegoAppDB in the report is for Gagan's steganography, not this module.)

## Tech stack

Python (scikit-learn, TensorFlow/PyTorch), TFLite, NumPy/Pandas, SHAP/LIME, Librosa/Scipy for
audio. Datasets are gitignored (`xai-engine/datasets/`, `*.csv`, `*.h5`, `*.model`) — download
locally, don't commit.
