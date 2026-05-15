# `<Brick Name>` brick

> Template. Copy to `docs/agent/bricks/<brick_name>.md` and fill in. Keep it **≤120 lines**, assertive, snippet-driven. No marketing prose. Use [web_ui.md](web_ui.md) and [motion_detection.md](motion_detection.md) as style references. Delete this blockquote in the final doc.

One-paragraph summary: what the brick does, what it's for, the package import path. Reference the source:
`https://github.com/arduino/app-bricks-py/tree/main/src/arduino/app_bricks/<brick>`.

## Install

Tell the user to add the brick from the App Lab GUI (Add Brick button). The App Lab edits `app.yaml` — never hand-edit `bricks:`. List any required sketch libraries (the user must add them via the App Lab GUI as well) and any pip packages the brick auto-pulls.

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
- Note anything the agent must tell the user to do via the App Lab GUI (bricks to add, sketch libraries to install).
