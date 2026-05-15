# `remote_sensor` peripheral

Hosts a WebSocket server on the board that accepts **one** remote client (a mobile app, a laptop, another device) sending IoT telemetry. Data is delivered to a Python callback as raw bytes. Source: `https://github.com/arduino/app-bricks-py/tree/main/src/arduino/app_peripherals/remote_sensor` (currently on PR branch `add-remote-sensor-peripheral`).

> Note: this is a **peripheral**, not a brick. Import path is `arduino.app_peripherals.remote_sensor`. App Lab still lists it in the Add Brick dialog — pick "Remote Sensor".

## Install

User must add the **Remote Sensor** peripheral from the App Lab GUI. No sketch library required. No pip deps to declare — the brick auto-pulls `websockets`.

## Python API

```python
from arduino.app_peripherals.remote_sensor import RemoteSensor

sensor = RemoteSensor(
    port=8080,
    timeout=3,
    use_tls=False,
    secret=None,             # set to a string to enable HMAC-SHA256 auth
    encrypt=False,           # True + secret → ChaCha20-Poly1305 payload encryption
    auto_reconnect=True,
)
```

| Method / property | Signature | Purpose |
| --- | --- | --- |
| `start` | `() -> None` | Bind the WebSocket server and start accepting clients. Idempotent. |
| `stop` | `() -> None` | Close the active client (sends a goodbye frame) and shut the server down. |
| `on_datapoint` | `(cb: Callable[[bytes], None])` | Called once **per WebSocket message**. The callback receives the BPP-decoded raw payload (`bytes`). **You parse it yourself** — typically `json.loads(data.decode())`. |
| `on_status_changed` | `(cb: Callable[[str, dict], None] \| None)` | Lifecycle events. `status ∈ {"disconnected","connected","streaming","paused"}`. `data` carries `client_address`, `client_name`. Pass `None` to unregister. |
| `status` | property → `str` | Current lifecycle status. |
| `url` | property → `str` | Full `ws[s]://ip:port` URL. |
| `ip` / `port` | property | Use to render pairing info / QR codes. |
| `security_mode` | property → `"none"\|"authenticated (HMAC-SHA256)"\|"encrypted (ChaCha20-Poly1305)"` | Debug. |

The class is also a context manager (`with RemoteSensor(...) as s: ...`).

## Wire protocol (what clients must send)

- Default: BPP-framed payloads. With `secret` set, payloads are HMAC-SHA256 authenticated. With `encrypt=True` they are ChaCha20-Poly1305 encrypted.
- Clients **may** send a `client_name=<name>` URL query param to identify themselves.
- Clients **may** opt out of BPP by appending `?raw=true` — **only when `secret is None`**, otherwise the flag is ignored.
- Payload body is application-defined. JSON is the conventional choice; CSV and raw binary work too — `on_datapoint` always gets `bytes`.

## Minimal example

```python
import json
import secrets
import string

from arduino.app_utils import App
from arduino.app_peripherals.remote_sensor import RemoteSensor


# 6-digit OTP, used as the HMAC pre-shared key. Show it in your UI / QR.
OTP = "".join(secrets.choice(string.digits) for _ in range(6))
print(f"Pairing OTP: {OTP}")

sensor = RemoteSensor(port=8080, secret=OTP)


def on_datapoint(raw: bytes) -> None:
    try:
        payload = json.loads(raw.decode())
    except Exception:
        return  # ignore non-JSON frames
    ax = payload.get("accelerometerX")
    ay = payload.get("accelerometerY")
    az = payload.get("accelerometerZ")
    print(f"accel: {ax}, {ay}, {az}")


def on_status(status: str, info: dict) -> None:
    print(f"sensor → {status} ({info})")


sensor.on_datapoint(on_datapoint)
sensor.on_status_changed(on_status)
sensor.start()

App.run()
```

## Gotchas

- `on_datapoint` receives **raw `bytes`**, not a parsed dict. Decode + parse yourself.
- The callback runs in a dedicated thread (`run_in_executor` on the sensor's asyncio loop). It is **not** a `Bridge.provide*` handler, so `Bridge.call(...)` from inside is **safe** (unlike `Bridge.provide` callbacks).
- Only **one client** is accepted at a time. New clients receive a JSON rejection and the socket is closed with code `1000` ("Server busy").
- Setting `encrypt=True` without a `secret` raises `RuntimeError`.
- `use_tls=True` + `encrypt=True` → `encrypt` is force-disabled (TLS already encrypts).
- The OTP / secret must be shared with the mobile client out-of-band — typically by displaying it (plus a QR code) on the WebUI.
- `sensor.ip` reflects the `HOST_IP` env var when set by App Lab; otherwise it is the bind address `"0.0.0.0"`. For QR codes prefer the value the *browser* sees (`window.location.hostname`) and use `sensor.port` only.
- The peripheral does not block `App.run()` — the server runs on its own thread/loop.
