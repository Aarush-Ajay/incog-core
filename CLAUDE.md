# Incog — Monorepo Context

Incog is a BMSCE CSBS capstone: a personal-safety Android app disguised as a fully functional
**calculator**. A hidden hardware gesture silently starts a covert "Ghost State" that captures
evidence, verifies the emergency with on-device + explainable AI, hides the evidence via
steganography, and dispatches SOS alerts — all with plausible deniability (nothing on screen
reveals the safety function).

This repo (`incog-core`) is a monorepo with **four module folders, one per teammate**. Each has
(or will have) its own `CLAUDE.md` with scoped detail. Work in your own module unless a
cross-module integration question comes up.

## The 12-phase pipeline

| Phase | What happens | Owner / module |
|---|---|---|
| 0 | Calculator decoy UI; background listener armed | Aarush · `mobile-client` |
| 1 | Hidden trigger detected: Volume Down-Down-Up ("DDU"), screen-off capable | Aarush · `mobile-client` |
| 2 | Ghost State: invisible foreground session starts | Aarush · `mobile-client` |
| 3 | Live sensor (accel/gyro/GPS) + mic-buffer collection | Aarush · `mobile-client` |
| 4 | Sensor fusion → feature vector (PeakAccel, MotionVariance, AudioEnergy, GPSVelocity) | Lipika · `xai-engine` |
| 5 | On-device TFLite inference → Prediction {label, confidence} | Lipika · `xai-engine` |
| 6 | Decision Engine: confidence vs threshold; else discard & return to Phase 0 | Lipika · `xai-engine` |
| 7 | Package evidence (audio + GPS + metadata) | Gagan · `security-vault` |
| 8 | AES-256-GCM encrypt | Gagan · `security-vault` |
| 9 | Fragment the encrypted blob | Gagan · `security-vault` |
| 10 | LSB-steganography-embed fragments into carrier PNGs | Gagan · `security-vault` |
| 11 | FastAPI backend: extract/decrypt/reassemble, store in PostgreSQL vault, async SHAP/LIME | Chirag · `c2-backend` |
| 12 | SOS Dispatcher: Twilio SMS/call/live-location to emergency contacts | Chirag · `c2-backend` |

## Team & module map

- **Aarush** — Mobile Core & Stealth Lead — `mobile-client/` — Phases 0–3. Native Kotlin/Compose.
- **Lipika** — AI & Intelligence Engineer — `xai-engine/` — Phases 4–6. Python (training) +
  on-device TFLite; server-side SHAP/LIME.
- **Gangandeep ("Gagan")** — Security & Steganography — `security-vault/` — Phases 7–10. Kotlin
  crypto (JCA/Tink) + OpenCV/LSB steganography.
- **Chirag** — Backend & SOS Dispatcher, Integration/API-contract lead — `c2-backend/` — Phases
  11–12. Python, FastAPI, PostgreSQL+PostGIS, Twilio.

### Inter-module handoff contracts
- Aarush → Lipika (Phase 3→4): raw sensor arrays + live audio buffer, as a data class
  (`SensorPacket`). **See `xai-engine/CLAUDE.md` for the full spec.**
- Lipika → Gagan (Phase 6→7): `EmergencyTriggered=true` signal + evidence metadata.
- Gagan → Chirag (Phase 10→11): multipart HTTPS upload of carrier `.png` stego images.

## Status (as of 2026-08-24)

- `mobile-client` — **Phases 0–3 complete and verified on-device (incl. locked-screen capture).**
  See `mobile-client/CLAUDE.md`.
- `xai-engine`, `security-vault`, `c2-backend` — not yet started (skeleton READMEs).

Everything lands on the `dev` branch. The repo has a branch-protection rule preferring pull
requests for changes.
