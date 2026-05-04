# Tier 3 — Per-camera HTTP control endpoint (architectural)

A medium-sized addition (~150–300 LoC) inside `vision_processor` that adds a small HTTP or TCP control endpoint per instance. This change addresses the protocol's underlying root cause directly, rather than working around symptoms.

## Background — the root cause

The current messaging design is a known anti-pattern: shared mutable state on a multicast bus with no ownership model ("blackboard with no librarian"). One `SSL_GeometryData` message contains the field plus all N cameras' calibrations; any process that publishes a wrapper packet must either republish all of it verbatim or destroy it.

The C++ binary chose to destroy it (`src/Perspective.cpp:48` calls `clear_calib()` before adding only its own camera's calib). That single design choice is the root cause of:

- The `clear_calib()` workaround dropping other cameras' calibs from the wire
- The defensive `calib_size() == 0` guard at `src/Perspective.cpp:57`
- The Python `geom_publisher`'s "absorb anything" listener loop (`python/geom_publisher.py:117-136`)
- The "command-as-state-absence" pattern where recalibration is triggered by removing a camera's calib from the bus

Tier 1 patches paper over symptoms. Tier 3 addresses the cause.

## The change

Add a small HTTP (or TCP/JSON-RPC) control endpoint inside each `vision_processor` instance.

### Suggested endpoints

| Endpoint | Operation |
|---|---|
| `GET /calib` | Return this camera's current calibration as JSON or protobuf |
| `POST /calib` | Replace this camera's calibration with the request body |
| `POST /calib/recalibrate` | Trigger field-line auto-calibration for this camera |
| `GET /calib/diagnostics` | Return the most recent auto-cal diagnostics (subsumes `TIER_2.md` #2) |
| `GET /frame.jpg` | Return the most recent debug frame snapshot (single-shot variant of `TIER_2.md` #1) |
| `GET /status` | Process state, frame rate, last error, geometry version, current config |

The endpoint binds to `localhost` only by default. No exposure to the broader network.

### Implementation outline

- Vendor a single-header HTTP library (e.g. `cpp-httplib`) — no new build dependency.
- Run the HTTP server on a dedicated thread, separate from the OpenCL processing loop.
- Cross-thread communication via existing mutex patterns in `Resources` and `Perspective`.
- Port number derived from `camId` (e.g. `8000 + camId`) or specified via CLI flag.

## Effect on the messaging design

With this endpoint:

- **Multicast becomes single-purpose**: detection frames only. The thing the multicast is actually good at — broadcasting one-to-many to teams' AI software. The geometry path stays in place for backwards compatibility but is no longer the primary control plane.
- **The blackboard pattern dissolves**: each `vision_processor` owns its own calibration and exposes it via its own endpoint. No more `clear_calib()` workaround. No more absorb-anything loops in the wrapper.
- **Commands become explicit**: "recalibrate camera 3" is a `POST /calib/recalibrate` to camera 3's endpoint, not a state-absence inference on the multicast.
- **The Python wrapper simplifies dramatically**: most "be careful about absorb loops" logic disappears. Wrapper makes REST calls per camera and renders responses. The `listener.py` task in the wrapper module layout (see `PYTHON_FEATURES.md`) becomes optional.

## Effort estimate

| Component | Approx LoC |
|---|---|
| HTTP server (single-header library, vendored) | 0 |
| Endpoint handlers and routing | 100–150 |
| State accessors (calib read/write, recalibrate trigger, diagnostics export) | 50–100 |
| Thread-safety glue with existing Resources/Perspective state | 30–50 |
| **Total** | **~150–300** |

## Tradeoffs

- **Backwards compatibility**: the existing multicast geometry protocol stays in place. Other ssl-vision-compatible tools continue to work. The control endpoint is purely additive.
- **Discovery**: the wrapper needs to know each `vision_processor`'s endpoint address. Simplest: each binary writes its port to a known PID file or registers in a small local discovery file. ~10 LoC. Or use a deterministic port per camera id.
- **Aggregation**: should there also be a centralized read-many endpoint (a "vision admin" service that aggregates per-camera states)? Out of scope for this tier; the wrapper can do that aggregation client-side.

## When this is worth doing

Tier 3 is worth doing if:

- The wrapper is intended to be more than a personal tool — i.e. usable by other SSL teams.
- The project owner wants to make the system genuinely easier to extend for other tooling.
- More than ~2 weeks of wrapper effort are otherwise planned (Tier 3 saves at least that much in wrapper defensive coding and is a one-time cost vs. recurring).

Tier 3 is **not** worth doing if the wrapper is a one-off personal tool with a short lifespan. In that case, Tier 1 patches plus a careful sole-publisher Python wrapper is sufficient.

## Relationship to other tiers

- Subsumes `TIER_2.md` #2 (auto-cal diagnostic export) — the diagnostics surface as `GET /calib/diagnostics`.
- Subsumes part of `TIER_2.md` #1 (debug-image stream) — single-shot snapshots via `GET /frame.jpg`. Continuous streaming for live preview is still served by the Tier 2 #1 sidechannel approach if desired.
- Compatible with `TIER_1.md` — the multicast tier-1 fixes still apply, since the multicast path remains.
