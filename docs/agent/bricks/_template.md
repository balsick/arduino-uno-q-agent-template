# `<Brick or Peripheral Name>`

> Template. Copy to `docs/agent/bricks/<name>.md` **or** `docs/agent/peripherals/<name>.md` and fill in. Keep it **≤120 lines**, assertive, snippet-driven. No marketing prose. Use [web_ui.md](web_ui.md) and [motion_detection.md](motion_detection.md) (bricks) or [../peripherals/remote_sensor.md](../peripherals/remote_sensor.md) and [../peripherals/websocket_camera.md](../peripherals/websocket_camera.md) (peripherals) as style references. Delete this blockquote in the final doc.

One-paragraph summary: what it does, what it's for, the package import path. Reference the source:
- brick → `https://github.com/arduino/app-bricks-py/tree/main/src/arduino/app_bricks/<name>`.
- peripheral → `https://github.com/arduino/app-bricks-py/tree/main/src/arduino/app_peripherals/<name>`.

## Install

- **Brick**: edit `app.yaml` directly to add `- arduino:<name>: {}` under `bricks:` (and any required device entries). List any required sketch libraries (add them to `sketch/sketch.yaml` under `libraries:` directly) and any pip packages the brick auto-pulls.
- **Peripheral**: no `bricks:` entry needed. Note any pip packages the peripheral auto-pulls. If it is consumed by a brick (e.g. a camera passed to a video brick), explain how to wire it.

## Python API

Document the constructor and the public methods you actually need. Real signatures only — get them from `__init__.py` / the main module of the brick, not from guesses.

```python
from arduino.app_bricks.<brick> import <ClassName>

obj = <ClassName>(arg1=..., arg2=...)
```

| Method | Signature | Purpose |
| --- | --- | --- |
| `method_a` | `(arg: type) -> return` | What it does. Note callback semantics if relevant. |
| `method_b` | `(name: str, cb: Callable)` | Register a callback. State threading / context. |

## Sketch-side pattern (only if the brick needs MCU input)

Short C++ snippet showing the typical sketch contract (e.g., sampling rate, `Bridge.notify` / `Bridge.provide_safe` calls, required library like `Modulino`). Skip this whole section for pure-Python bricks.

```cpp
#include <Arduino_RouterBridge.h>
// ...
```

## Minimal example

Full `python/main.py` example, runnable as-is. End with `App.run()` as the last line.

```python
from arduino.app_utils import App, Bridge
from arduino.app_bricks.<brick> import <ClassName>

obj = <ClassName>()

# wire up callbacks / Bridge handlers here

App.run()
```

## Gotchas

- List concrete pitfalls: type mismatches, default ports, missing files, deadlock conditions, threading rules.
- Note any required env vars, model assets, or external services.
- Note anything the agent must declare in `app.yaml` (bricks, device bindings) or `sketch/sketch.yaml` (libraries) — the agent now edits these files directly, no GUI step.
