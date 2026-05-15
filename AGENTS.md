# AGENTS.md — Arduino App Lab project (UNO Q)

This repository is an **Arduino App Lab** application targeting the **Arduino UNO Q** (dual-processor: Linux MPU + Zephyr MCU). Python runs the high-level logic on Linux; the C++ sketch runs real-time logic on the microcontroller; they talk over the **Bridge** (RPC). Bricks are pre-packaged services (Docker containers) reused via Python APIs.

You (the agent) MUST follow this file. It is the source of truth.

---

## 1. When to read what

This file is always in context. The files under `docs/agent/` are **not**: load them on demand based on the task.

| Task | Load |
| --- | --- |
| Anything involving Python ↔ MCU communication | [docs/agent/bridge.md](docs/agent/bridge.md) |
| Build / serve a Web UI (HTML/CSS/JS, REST, WebSocket) | [docs/agent/bricks/web_ui.md](docs/agent/bricks/web_ui.md) + [docs/agent/frontend.md](docs/agent/frontend.md) |
| Detect motion / gestures from an accelerometer | [docs/agent/bricks/motion_detection.md](docs/agent/bricks/motion_detection.md) |
| Receive telemetry from a phone / remote client over WebSocket | [docs/agent/bricks/remote_sensor.md](docs/agent/bricks/remote_sensor.md) |
| Persist data on the board in a SQLite DB | [docs/agent/bricks/dbstorage_sqlstore.md](docs/agent/bricks/dbstorage_sqlstore.md) |
| CLI / running / logs / ADB / SSH | [docs/agent/cli.md](docs/agent/cli.md) |
| Deploy / run the app on the board (from a dev machine or on-board) | [docs/agent/cli.md](docs/agent/cli.md) |
| Use a brick not yet documented here | Follow the **new-brick discovery workflow** in section 8 of this file. |

If a doc is missing for a feature you need, follow §8 to author it (or write it from the template), then use it.

---

## 2. Project layout (fixed)

```
AGENTS.md                # this file
CLAUDE.md                # symlink to AGENTS.md
README.md                # user-facing project description (optional)
app.yaml                 # agent-editable; declares app name, ports, bricks
python/
  main.py                # entrypoint; App.run() MUST be the last line
  requirements.txt       # pip deps (this file lives under python/, NOT root)
sketch/
  sketch.ino             # C++ for the MCU (optional)
  sketch.yaml            # agent-editable; declares sketch libraries
assets/                  # WebUI static root (served at /); needs index.html
  index.html
  css/
    app.css
  js/
    app.js
    vendor/              # third-party JS, downloaded (no CDN at runtime)
      socket.io.min.js
      VERSIONS.md        # pinned versions + source URLs
docs/agent/              # on-demand docs for you
  bridge.md
  cli.md
  frontend.md
  bricks/                # one file per brick (import: arduino.app_bricks.<name>)
    _template.md
    web_ui.md
    motion_detection.md
    dbstorage_sqlstore.md
  peripherals/           # one file per peripheral (import: arduino.app_peripherals.<name>)
    remote_sensor.md
    websocket_camera.md
```

The board stores deployed apps under `/home/arduino/ArduinoApps/<app-name>/`.

---

## 3. Absolute rules

1. **`App.run()` is the last line of `python/main.py`.** Anything after it is ignored by the framework.
2. **You own `app.yaml` and `sketch/sketch.yaml`.** Create them when missing and edit them directly — including the `bricks:` list in `app.yaml` and the `libraries:` list in `sketch/sketch.yaml`. Add whatever bricks/libraries the requested feature requires. Do not tell the user to add them through the App Lab GUI.
3. In C++, expose callbacks with **`Bridge.provide_safe(name, fn)`** (executes in loop context). Reserve `Bridge.provide` for advanced use.
4. **Do NOT call `Bridge.call()` or `Monitor.print()` inside a callback exposed via `Bridge.provide*`.** This deadlocks the Bridge.
5. **No blocking `delay()` / `sleep()` longer than a few ms** in either loop — the Bridge needs to service requests.
6. Bridge argument types must match exactly between Python and C++ (`int` ↔ `int`, `float` ↔ `float`, `const char*` ↔ `str`). Mismatches fail silently.
7. `Arduino_RouterBridge` is included by default on Zephyr core ≥ 0.55.0 for UNO Q. Do NOT add it to `sketch.yaml`. Just `#include <Arduino_RouterBridge.h>`.
8. Python dependencies go in **`python/requirements.txt`** only. Never put one in the repo root.
9. `WebUI` requires `assets/index.html` at the configured `assets_dir_path` (default `/app/assets`, which corresponds to `assets/` in the repo). Without it, the brick fails to start.
10. Third-party JS/CSS is **downloaded into `assets/js/vendor/` (or `assets/css/vendor/`)** and referenced via relative paths. **No `<script src="https://cdn...">`** at runtime.
11. Write code comments in English. All documentation in this repo is in English.

---

## 4. Python skeleton (`python/main.py`)

Two valid shapes — pick one, never mix.

**With a periodic loop:**

```python
from arduino.app_utils import App, Bridge
import time

def loop():
    # runs repeatedly on the Linux side
    Bridge.call("blink_led")  # call an MCU function
    time.sleep(1)

App.run(user_loop=loop)
```

**Event-driven only (UI / Bridge callbacks drive the app):**

```python
from arduino.app_utils import App, Bridge

def on_mcu_event(value: int):
    print(f"MCU said: {value}")

Bridge.provide("on_mcu_event", on_mcu_event)

App.run()
```

See [docs/agent/bridge.md](docs/agent/bridge.md) for the full API.

---

## 5. Sketch skeleton (`sketch/sketch.ino`)

```cpp
#include <Arduino_RouterBridge.h>

void blink_led() {
  digitalWrite(LED_BUILTIN, !digitalRead(LED_BUILTIN));
}

void setup() {
  Monitor.begin();
  pinMode(LED_BUILTIN, OUTPUT);
  Bridge.begin();
  Bridge.provide_safe("blink_led", blink_led);   // callable from Python
}

void loop() {
  // Keep the loop non-blocking. Use Bridge.notify(...) to push data to Python.
}
```

See [docs/agent/bridge.md](docs/agent/bridge.md) for `Bridge.call`, `Bridge.notify`, argument passing, and bidirectional patterns.

---

## 6. WebUI skeleton (Python side)

```python
from arduino.app_utils import App, Bridge
from arduino.app_bricks.web_ui import WebUI

ui = WebUI()  # default port 7000, serves ./assets/ as static root

ui.expose_api("GET", "/api/status", lambda: {"ok": True})

def on_command(sid, data):
    Bridge.call("apply_command", int(data["value"]))

ui.on_message("command", on_command)

def push_telemetry(value: int):
    ui.send_message("telemetry", {"value": value})

Bridge.provide("push_telemetry", push_telemetry)

App.run()
```

For the HTML/CSS/JS side (layout, Socket.IO client, vendoring, visual style) read [docs/agent/frontend.md](docs/agent/frontend.md). For the full brick API read [docs/agent/bricks/web_ui.md](docs/agent/bricks/web_ui.md).

---

## 7. Standard workflow per task

When the user asks for a feature:

1. **Decide which bricks are needed.** For each brick, ensure `docs/agent/bricks/<brick>.md` exists. If not, follow the new-brick discovery workflow in §8 before proceeding.
2. **Decide if the MCU is involved.** If yes, load `docs/agent/bridge.md` and write/extend `sketch/sketch.ino`.
3. **Decide if there is a UI.** If yes, load `docs/agent/frontend.md` and `docs/agent/bricks/web_ui.md`; write/update `assets/index.html`, `assets/css/app.css`, `assets/js/app.js`; download any new JS dependency into `assets/js/vendor/` and record it in `assets/js/vendor/VERSIONS.md`.
4. **Update Python**: edit `python/main.py`, append packages to `python/requirements.txt`.
5. **Update `app.yaml` and `sketch/sketch.yaml` yourself.** Add the bricks you used to `app.yaml` under `bricks:` (e.g. `- arduino:web_ui: {}`). Add any sketch libraries you used to `sketch/sketch.yaml` under `libraries:` (e.g. `- Modulino`). Keep `name`, `description`, `ports`, `icon` in `app.yaml` sensible.
6. **Deploy & run.** Use `arduino-app-cli` on the board, or `adb` from your dev machine — see [docs/agent/cli.md](docs/agent/cli.md). Detect your execution context (on-board vs off-board) before issuing commands.
7. Keep changes minimal and focused — no speculative refactors.

---

## 8. New-brick / peripheral discovery workflow

When the user asks for a feature that requires a brick or peripheral **not yet documented** in `docs/agent/bricks/` or `docs/agent/peripherals/`, do this — do not skip steps.

> Bricks live under `arduino.app_bricks.<name>` and are declared in `app.yaml` under `bricks:`. Peripherals live under `arduino.app_peripherals.<name>` and are used directly from Python (no `bricks:` entry). Some peripherals are consumed by bricks (e.g. a `WebSocketCamera` passed to `VideoObjectDetection`) — in that case the brick that uses it still goes in `app.yaml`.

1. **Scrape sources.** Fetch:
   - `https://github.com/arduino/app-bricks-examples` — search the repo for files that import or use the target (look in `examples/*/python/main.py`, the `app.yaml`, and the example READMEs). Read the most relevant examples end-to-end.
   - Source for the implementation:
     - brick → `https://github.com/arduino/app-bricks-py/tree/main/src/arduino/app_bricks/<name>` (README + `__init__.py` + `examples/`).
     - peripheral → `https://github.com/arduino/app-bricks-py/tree/main/src/arduino/app_peripherals/<name>` (README + `__init__.py` + module files + `examples/`).
2. **Draft the doc locally.** Copy `docs/agent/bricks/_template.md` to either `docs/agent/bricks/<name>.md` or `docs/agent/peripherals/<name>.md`. Fill it in: ≤120 lines, assertive, snippet-driven, no marketing prose. Match the style of `web_ui.md`, `motion_detection.md`, `peripherals/remote_sensor.md`, `peripherals/websocket_camera.md`.
3. **Ask the user to confirm.** Show: name, kind (brick vs peripheral), sources scraped (URLs), draft size, and a one-line summary of the API surface. Wait for an explicit yes/no before writing anything.
4. **On yes — write into the current project:**
   - Save the doc under the correct folder (`bricks/` or `peripherals/`).
   - Add a row to the table in §1 of this `AGENTS.md`.
5. **Open a PR upstream so the template grows.** The template repo is `balsick/arduino-uno-q-agent-template`. From a working clone/fork:
   ```bash
   gh repo fork balsick/arduino-uno-q-agent-template --clone --remote=false || true
   # in your local fork:
   git checkout -b docs/<name>
   # add docs/agent/{bricks|peripherals}/<name>.md and the AGENTS.md §1 table entry
   git add docs/agent AGENTS.md
   git commit -m "docs: add <name> (<brick|peripheral>)"
   git push -u origin docs/<name>
   gh pr create --repo balsick/arduino-uno-q-agent-template \
     --title "docs: add <name> (<brick|peripheral>)" \
     --body  "Adds agent docs for the \`<name>\` <brick|peripheral>. Sources: <URLs scraped>."
   ```
   Tell the user the PR URL.
6. **On no — abort.** Do not write the doc. The agent can still proceed (referring to the scraped sources inline) but should warn the user that future sessions will lack the local doc.

---

## 9. Constraints checklist (run before declaring done)

- [ ] `App.run()` is the last line of `python/main.py`.
- [ ] No `Bridge.call` / `Monitor.print` inside `Bridge.provide*` callbacks.
- [ ] No long `delay()` / `sleep()`.
- [ ] Argument types match across Bridge calls.
- [ ] `python/requirements.txt` updated if new pip deps were used.
- [ ] If WebUI is used: `assets/index.html` exists, JS vendor libs are local (not CDN).
- [ ] `app.yaml` lists every brick the code uses, under `bricks:`.
- [ ] `sketch/sketch.yaml` lists every sketch library the code uses, under `libraries:`.
- [ ] For every brick/peripheral used, the matching doc exists under `docs/agent/bricks/` or `docs/agent/peripherals/` (else §8 was followed).
- [ ] Deployment path chosen correctly: on-board `arduino-app-cli` vs off-board `adb shell arduino-app-cli` (see `docs/agent/cli.md`).
