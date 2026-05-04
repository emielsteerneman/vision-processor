# Python Wrapper Feature Load Estimate

Estimated implementation/cognitive load per feature for the planned Python web wrapper around `vision_processor`. The wrapper replaces the existing `geom_publisher.py`, manages `vision_processor` subprocesses, and provides a browser UI for camera calibration and configuration.

Load scale:

- **Low**: standard library/framework patterns, well-trodden path. ~hours of work.
- **Med**: requires care, has known gotchas you can plan around. ~days of work.
- **High**: novel implementation, math/correctness risk, debugging unknowns. ~week+ of work.

## Feature table

| Feature | Load | Why |
|---|---|---|
| Start / stop / restart `vision_processor` subprocess | Low | Standard subprocess management. SIGTERM cleanup is already implemented in the binary. |
| Show stderr & process state in UI | Low | Read process pipe, push lines over WebSocket. |
| Show `sample.png` in browser, auto-refresh on disk change | Low | File watcher + HTTP route. |
| Edit `config.yml` in a form, save, restart | Low | Form fields backed by YAML keys. |
| Comment-preserving YAML writes (old values as comments) | Low | One library choice (`ruamel.yaml`). |
| Atomic file save (temp file + rename) | Low | ~5-line standard idiom. |
| Click-to-set 4 field corners on sample image | Low–Med | Canvas with 4 draggable handles, write pixel coordinates to YAML. Bounded frontend work. |
| Replace `geom_publisher.py` (read YAML, publish at 1 Hz on multicast) | Low | The existing script is 142 lines doing exactly this. Port into the app. |
| Listen on multicast & absorb auto-cal as initial seed | Low | When a packet arrives with calib for camera N and we have none, copy values into in-memory state. |
| Slider / number input UI for camera world position (extrinsics) | Low | HTML inputs bound to in-memory values. |
| Push calibration changes over the network as the user edits | Low | Re-uses the publisher; mark in-memory state dirty, publisher re-emits. |
| **Live reprojected field-lines overlay on the image** | **High** | Translate C++ camera + distortion math to Python (10-iteration distortion solve plus quaternion transforms), or compile a Python binding. Carries silent-drift risk over time. The differentiated feature `ssl-vision` cannot do. |
| Single multiplexed WebSocket | Low | FastAPI + JSON envelope, one endpoint. |
| Concurrency hygiene (one `asyncio.Lock` + immutable state snapshots) | Low | One pattern applied consistently. |
| Multi-camera (N subprocesses, N calib entries in shared state) | Low–Med | Happy path is just an array. Edge cases (camera crash mid-edit, simultaneous saves) are not day-1 work. |
| Crash isolation: split publisher into separate process | Med | Optional. Operational hardening for tournament use. Skip for MVP. |

## Notes

The reprojection overlay is the only inherently expensive feature. Cheap variants exist:

- **Cheap path**: skip the overlay. Sliders update calibration over the network; user opens `python/cam_viewer.py` (mpv) in another window for live visual feedback. Browser shows the static sample image only.
- **Medium path**: render field lines as straight segments, ignoring distortion. Wrong at edges, fine in the middle. Useful for rough alignment, misleading for fine-tuning.
- **Full path**: properly distorted reprojection. The differentiated feature.

If C++ change budget is available, see `TIER_2.md` — the debug-image stream addition makes the *image underneath* the overlay update live during slider drag, dramatically increasing the value of the overlay feature.

## Architectural posture

The planned wrapper is a single Python process that:

- Holds one in-memory `SSL_GeometryData` (field + N calibs keyed by `camera_id`) as the runtime source of truth
- Loads it from `geometry.yml` on startup, writes back on user save
- Publishes it on the multicast at 1 Hz (heartbeat) plus on-change burst (debounced ~100 ms)
- Replaces `geom_publisher.py` entirely — wrapper is the sole publisher of `SSL_GeometryData` while running
- Supervises N `vision_processor` subprocesses (one per camera)
- Exposes a FastAPI app with REST + a single multiplexed WebSocket for the browser SPA
- Headless mode (`--headless` / `--no-ui`) skips the FastAPI app and runs as a `geom_publisher` replacement only

Concurrency: FastAPI handlers and async tasks share the asyncio event loop. One `asyncio.Lock` around state mutations, with an immutable-snapshot pattern (mutators produce a new dataclass, atomic swap under lock). Publisher reads a snapshot reference without lock.

Module layout:

```
wrapper/
  __main__.py        # CLI parse, --headless toggle, uvicorn boot
  app.py             # FastAPI app factory
  state.py           # In-memory SSL_GeometryData + config state
  publisher.py       # 1 Hz heartbeat + on-change burst
  listener.py        # Multicast subscribe, seed-on-cold-start absorb policy
  supervisor.py      # vision_processor subprocess management
  yaml_io.py         # ruamel.yaml comment-preserving I/O
  api/{rest,ws}.py
  proto/             # generated from project's existing proto/
  static/            # bundled SPA
```
