# Tier 1 — Tiny upstream bug-fix patches

Protocol-level fixes for actual defects identified during code review of the messaging layer. Each is ~1–5 lines. These are bug fixes, not feature additions, and worth doing regardless of any wrapper plans.

## Patches

### 1. Set source on geometry republishes

**Files**: `src/Perspective.cpp:46-50`, `src/calib/GeomModel.cpp:582-586`

The C++ binary republishes wrapper packets containing geometry in two paths:

- After auto-calibration completes (`GeomModel.cpp:582-586`)
- When completing a partial calibration with derived world-frame fields (`Perspective.cpp:46-50`)

Neither call sets `wrapper.set_source(...)`. The detection-frame publish at `src/main.cpp:343` does set source. The geometry republishes do not.

**Patch**: add `wrapper.set_source(SSL_SOURCE_VISION_PROCESSOR);` to both republish sites.

**Effect**: lets any consumer disambiguate "geometry packet from a `vision_processor` republish" vs. "geometry packet from the geometry publisher." Eliminates ~80% of the absorb-loop hazard when building tooling around the multicast bus.

### 2. Deterministic serialization for change detection

**Files**: `src/udpsocket.cpp:117`, `src/udpsocket.cpp:153`

`VisionSocket::geometryCheck` and `VisionSocket::parse` use `MessageDifferencer::Equals` to detect changes in received `SSL_GeometryData`. With float fields and non-deterministic protobuf serialization, byte-equal payloads can compare unequal after a roundtrip, causing spurious `geometryVersion` bumps.

`python/geom_publisher.py:124` already learned this lesson and uses `SerializeToString(deterministic=True)` for its own equality check.

**Patch**: switch C++ change-detection to compare `SerializeAsString` outputs with deterministic serialization, or add a monotonic version field on the wrapper and compare that instead.

**Effect**: eliminates spurious version bumps. Each spurious bump triggers `Perspective::geometryCheck` → `image2field` recompute over the full image (`Perspective.cpp:75-88`). Real CPU saved on every camera, every cycle. Also removes the load-bearing assumption behind the defensive guard at `Perspective.cpp:57`.

### 3. `geom_publisher.py` camera_id index bug

**File**: `python/geom_publisher.py:127`

```python
calib[camera.camera_id].CopyFrom(camera)
```

This uses `camera_id` as a **list index**, not a lookup key. Works only when camera ids are 0, 1, 2, ... in order with no gaps.

A 4-camera setup with ids `[0, 1, 3, 5]` would write camera id 5's calib into list slot 5, which doesn't exist (IndexError) or overwrites the wrong slot if it does.

**Patch**: iterate `calib` and match by the `camera_id` field, not by list index.

**Effect**: removes a silent corruption bug for non-contiguous camera id setups.

## Why these are Tier 1

These are bug fixes for actual defects, not feature additions. Each reduces cognitive load on every other tool that touches this protocol — including any planned Python wrapper. They are small enough to land as upstream PRs and benefit the project independently of the wrapper effort.

Recommended order:

1. Patch #1 (set_source) — biggest immediate win for tooling
2. Patch #3 (geom_publisher index) — silent bug fix
3. Patch #2 (deterministic equality) — performance + correctness, but more invasive
