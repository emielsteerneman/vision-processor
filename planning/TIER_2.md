# Tier 2 — Small additions that unlock high-value features

C++ additions of ~50–100 LoC each. Each turns a previously-hard wrapper feature into an easy one, or attacks an underlying project pain point that no wrapper alone can fix.

## Additions

### 1. Debug-image stream mode

**Where**: `src/main.cpp` per-frame loop. Currently writes `.sample.{camId}.png` exactly once at frame 100 (line 399), then never again.

After the cold-start sample, no image is exposed to disk for the lifetime of the process. The RTP livestream cycles four debug views every 20 seconds (`src/main.cpp:378-391`) but is heavyweight (encoded H.264 over UDP) and not friendly to a browser-side preview at low rates.

**Change**: add a CLI flag (e.g. `--debug-stream <interval_frames>` or `--debug-snapshot-path <dir>`) that makes the binary write a JPEG of the current quad-channel composite (`r.quad2rgba(channels)`) every N frames. Alternatively stream JPEG-over-Unix-domain-socket on a sidechannel for in-memory delivery.

**Effort**: ~50 LoC. The render path already exists; the existing `quad2rgba(channels)->save(...)` at `main.cpp:399` is the prototype. This change taps the same data into a slower, lighter sidechannel.

**Unlocks (in `PYTHON_FEATURES.md`)**:

- **Live preview during slider drag.** The reprojection overlay can be drawn on top of an image that *updates* as calibration changes, instead of a frozen cold-start sample. This dramatically increases the value of the (already-expensive) overlay feature. Without this, the overlay only makes sense on the static cold-start sample.
- **Live diagnostic visibility during config tuning** (gain, exposure, white balance) without restarting the binary.

### 2. Auto-calibration diagnostic export

**Where**: `src/calib/GeomModel.cpp` (auto-cal path) and `src/calib/LineDetection.cpp`.

Auto-calibration is the project's #1 unsolved pain point per the project author. The binary computes residuals, model error (`GeomModel.cpp:579`), and intermediate hypotheses internally but exposes none of them to the outside.

**Change**: write auto-cal diagnostic state to a structured file (JSON or protobuf) on each auto-cal run, or expose via a small status endpoint:

- Detected line corners and their residuals
- Model error progression across iterations
- Final cost-function value
- Reprojection error per detected line
- Final calibration parameters (so consumers can confirm what the binary actually decided)

**Effort**: ~50–100 LoC depending on output format. The data already exists internally; this is a serialization addition.

**Why this matters strategically**: a wrapper that lets users *manually fix* bad auto-cal results is helpful but not transformative — the underlying instability remains. Exposing diagnostics is the prerequisite for understanding, and eventually fixing, the instability. This is the "diagnostic-first phasing" worth considering before pouring effort into the manual-edit UX.

### 3. Hot config reload

**Where**: `src/Resources.cpp` (config loading path) and per-driver code under `src/driver/`.

Today every `config.yml` change requires killing and restarting the binary. The geometry path already supports runtime updates via the multicast (`r.socket->geometryCheck()` runs every frame). Other config fields (thresholds, camera params, color references) require restart.

**Change**: file-watch `config.yml`. On change, reparse and re-apply runtime-tunable fields:

- Thresholds and color references — trivial to hot-swap (just update the floats/vectors in `Resources`).
- Camera-driver-level fields (exposure, gain, white_balance) — need a driver-specific reapply path per backend.
- `line_corners` — already triggers re-calibration via the geometry path; should work without further change once Tier 1 patches are applied.

**Effort**: ~50 LoC for the file watch + reapply logic for thresholds/colors. Camera-driver fields add ~50 LoC per backend (Spinnaker, mvIMPACT, OpenCV).

**Unlocks**: kills the "edit YAML → restart binary" cycle entirely. Combined with the wrapper's UI editing, makes calibration tuning a closed-loop interaction with no restart in the loop.

## Selection guidance

If only one Tier 2 addition is taken: **#2 (auto-cal diagnostic export)** has the highest project-strategic value. It attacks the actual root pain (auto-cal instability) instead of working around it.

If two are taken: add **#1 (debug-image stream)**. Live preview is the differentiating UX feature for the wrapper and turns the High-load reprojection overlay into a meaningfully better experience.

**#3 (hot config reload)** is the highest-effort of the three and the most invasive to existing code paths. Justify only if reduced-restart is a primary goal beyond what the wrapper alone provides.
