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

### 2. Deterministic serialization for change detection — INVESTIGATED, DROPPED

**Files**: `src/udpsocket.cpp:117`, `src/udpsocket.cpp:153`

Initial framing was: `MessageDifferencer::Equals` is non-deterministic and float-sensitive, causing spurious `geometryVersion` bumps from serialization noise.

On reading the actual code, this turned out to be misdiagnosed. `MessageDifferencer::Equals` does **field-level** comparison, not byte-level. For float fields it compares the deserialized float values via `==`. Switching to deterministic-`SerializeAsString`-byte-comparison would not change behaviour for this codebase (proto2, no maps, no presence ambiguity).

The underlying observation about float drift via the quaternion roundtrip in `getProto()` (non-associative IEEE-754 in `f2iOrientation * -pos` and its inverse) is real — `vision_processor` republishes carry slightly different float bits than the original calib it received. But `MessageDifferencer::Equals` correctly detects that drift as "different" because the float values genuinely differ. Switching the comparator does not suppress it.

The actual fixes for the version-churn would be either:
- Epsilon-based float comparison (semantic change, debatable)
- A monotonic version field on the wrapper (proto change — a bigger PR than belongs in Tier 1)
- Stop the upstream cause of drift entirely, by filtering out `vision_processor` republishes

Patch 1 (set_source on republishes) addresses the wrapper-builder pain by making republishes filterable. Within `vision_processor` itself, the self-loop drift remains a long-standing inefficiency, but it is not a regression and is out of scope for a tier-1-style PR.

This patch is therefore not pursued.

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

Patches actually pursued:

1. Patch #1 (set_source) — biggest immediate win for tooling
2. Patch #3 (geom_publisher index) — silent bug fix

Patch #2 (deterministic equality) was investigated and dropped — see its section above.
