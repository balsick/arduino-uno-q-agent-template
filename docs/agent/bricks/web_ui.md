# `web_ui` brick

Embeddable web server (FastAPI + Uvicorn + Socket.IO) that serves a static frontend, REST APIs, and a WebSocket channel. Source: `https://github.com/arduino/app-bricks-py/tree/main/src/arduino/app_bricks/web_ui`.

## Install

User must add the **WebUI - HTML** brick from the App Lab GUI. App Lab edits `app.yaml` — don't hand-edit. No sketch library needed (the brick is pure Python).

## Python API

```python
from arduino.app_bricks.web_ui import WebUI

ui = WebUI(
    addr="0.0.0.0",
    port=7000,                       # default
    ui_path_prefix="",
    api_path_prefix="",
    assets_dir_path="/app/assets",   # maps to ./assets/ in the repo
    cors_origins="*",
    use_tls=False,
)
```

| Method | Signature | Purpose |
| --- | --- | --- |
| `expose_api` | `(method: str, path: str, function: Callable)` | Register a REST route. `function` is FastAPI-style (returns dict/JSON). |
| `on_message` | `(message_type: str, cb: Callable[[sid, data], Any])` | Handle a Socket.IO event. If `cb` returns a value, the server emits `<message_type>_response` back to the same `sid`. |
| `send_message` | `(message_type: str, message: dict\|str, room: str\|None=None)` | Push a Socket.IO event to all clients (or a specific room / sid). |
| `on_connect` | `(cb: Callable[[sid], None])` | Called when a client connects. |
| `on_disconnect` | `(cb: Callable[[sid], None])` | Called when a client disconnects. |
| `expose_camera` | `(path: str, camera: BaseCamera, jpeg_quality: int=80)` | Mount an MJPEG stream at `path` (consume with `<img src="..."/>`). |
| `local_url` / `url` | (property) | Resolved URLs (local vs. external). |

Notes:
- Static files are served at the root: `assets/index.html` → `/`, `assets/js/app.js` → `/js/app.js`, etc.
- Socket.IO is mounted at `/socket.io` (client uses `io()` with same origin).
- The brick uses CORS `*` by default; tighten via `cors_origins`.
- Files inside `assets/` are served with `Cache-Control: no-store` — safe to redeploy.

## Hard requirement

`assets/index.html` **must** exist or the brick raises `RuntimeError` on startup. The directory is `assets_dir_path` (default `/app/assets` inside the container, which is `assets/` in the repo).

## Minimal example — REST + WebSocket

```python
from arduino.app_utils import App, Bridge
from arduino.app_bricks.web_ui import WebUI

ui = WebUI()

# REST
state = {"led": False}
ui.expose_api("GET", "/api/state", lambda: state)

# WebSocket inbound (from browser)
def on_command(sid, data):
    state["led"] = bool(data.get("on"))
    Bridge.call("set_led", 1 if state["led"] else 0)
    return {"ok": True}      # auto-emitted as "command_response" to this sid

ui.on_message("command", on_command)

# WebSocket outbound (from Python, e.g., MCU telemetry)
def push_sample(value: int):
    ui.send_message("telemetry", {"value": value})

Bridge.provide("push_sample", push_sample)

App.run()
```

Frontend wiring lives in `assets/`. See [../frontend.md](../frontend.md) for the HTML/CSS/JS conventions and how to vendor `socket.io.min.js` locally.

## Gotchas

- **No CDN scripts** — vendor `socket.io.min.js` into `assets/js/vendor/` (see [../frontend.md](../frontend.md)).
- Default port `7000`; change with `WebUI(port=...)` only if it clashes.
- Don't call `Bridge.call(...)` inside a Python callback that is itself exposed via `Bridge.provide(...)` (deadlock). It is fine to call `Bridge.call(...)` inside `ui.on_message(...)` handlers and from `expose_api` handlers — those run in the WebUI brick, not in the Bridge thread.
- `send_message` is thread-safe but the brick's asyncio loop must be running; calling it before `App.run()` no-ops with a debug log.
- Returning a value from an `on_message` callback emits `"<event>_response"` to that `sid` only. Use `send_message` for broadcast.
- If you need TLS, set `use_tls=True` and provide certs in `/app/certs`.
