# Tier 4 — Three-channel protocol redesign (NOT RECOMMENDED)

A full architectural overhaul replacing the current single-message geometry protocol with three logically distinct channels and explicit ownership semantics.

## What it would look like

Three independent message streams on three multicast groups (or three port numbers on one group):

1. **Field geometry channel** — single writer (`geom_publisher` or its replacement). Carries `SSL_GeometryFieldSize` and `SSL_GeometryModels` only. Versioned with a monotonic counter.
2. **Per-camera calibration channels** — one channel per camera, each owned exclusively by that camera's `vision_processor`. Carries only `SSL_GeometryCameraCalibration` for that camera. Independently versioned.
3. **Command channel** — addressed messages (target `camera_id`) with response acknowledgements. Used for "recalibrate," "use this calib as starting hypothesis," "freeze auto-cal," etc.

The detection-frame multicast (`224.5.23.2:10006` for `SSL_DetectionFrame`) is left unchanged — it's already correctly designed (single-writer-per-topic by construction, one publisher per camera).

## Why this is the architecturally correct design

- Each piece of state has a unique, named owner.
- Commands are first-class, not inferred from state absence.
- Updates to one camera's calibration don't require knowing about other cameras' calibrations.
- The "blackboard with no librarian" pattern is eliminated entirely.
- Versioning is explicit and per-channel, not based on float-equality of an entire combined message.

This is the design the system arguably should have had from the start, given how distributed it is.

## Why it is NOT recommended

1. **Wire-format compatibility.** The SSL community has converged on `SSL_WrapperPacket{geometry: SSL_GeometryData{field, calib[], models}}` as the canonical format. Other teams' AI software consumes this directly via the same multicast group used for detection. A protocol redesign isolates the project from the broader ecosystem unless every downstream consumer migrates simultaneously.

2. **Effort vs. benefit.** Implementation effort is several weeks across C++, Python, and any documentation/migration tooling. The same practical outcome (clear ownership, explicit commands, sane wrapper integration) can be achieved in days by adding the per-camera HTTP endpoint described in `TIER_3.md` while keeping the existing multicast for backwards compatibility.

3. **Project ownership.** This change is too invasive to land as a contributor PR. It would require buy-in from the original project author, coordinated migration, and ongoing maintenance commitment. The author has indicated the existing protocol "has been pretty much sufficient" and that additional interfaces are not currently necessary.

4. **The actual project pain is auto-calibration stability, not protocol design.** The protocol is bad-but-workable; auto-cal is unsolved. Engineering investment is better directed at `TIER_2.md` #2 (auto-cal diagnostics) than at this overhaul.

## When this might be worth revisiting

- A new SSL vision standard emerges that explicitly supports per-camera channels.
- The number of independent tools needing to write to the geometry bus grows beyond two or three (current: just `geom_publisher` and `vision_processor` republishes).
- The project diverges from `ssl-vision` ecosystem compatibility for unrelated reasons (e.g. moves to a non-multicast transport).

Until any of those happen, `TIER_3.md` captures essentially all the architectural benefit for a fraction of the cost and risk.

## Documented for completeness only

This document exists so the option is recorded and can be revisited. It is not an endorsement and is not on any near-term roadmap.
