# `websocket_camera` peripheral

Hosts a WebSocket server on the board that accepts **one** remote client (a phone, laptop, another device) streaming JPEG/PNG/WebP/BMP/TIFF frames. The peripheral exposes a `Camera`-compatible interface, so it plugs directly into bricks that consume a camera object (e.g. `VideoObjectDetection`, `VideoFaceDetection`). Source: `https://github.com/arduino/app-bricks-py/tree/main/src/arduino/app_peripherals/camera` (class `WebSocketCamera` in `websocket_camera.py`).

> This is a **peripheral**, not a brick. Import path is `arduino.app_peripherals.camera`.

## Install

No `bricks:` entry — peripherals are not bricks. Use it directly from Python. If you feed it into a brick that requires a `remote_camera_*` device (e.g. `video_objectdetection`), declare the device under that brick in `app.yaml`:

```yaml
bricks:
  - arduino:video_object_detection:
      devices:
        - remote_camera_0
  - arduino:web_ui: {}
```

No sketch library required. No extra pip deps to declare — the peripheral pulls `websockets`, `opencv-python`, `numpy` automatically.

## Python API

```python
from arduino.app_peripherals.camera import WebSocketCamera

camera = WebSocketCamera(
    port=8080,
    timeout=3,
    use_tls=False,
    secret=None,                # set to a string to enable HMAC-SHA256 auth
    encrypt=False,              # True + secret → ChaCha20-Poly1305 payload encryption
    resolution=(640, 480),
    fps=10,
    adjustments=None,           # optional frame transform pipeline
    auto_reconnect=True,
)
```

| Method / property | Signature | Purpose |
| --- | --- | --- |
| `start` | `() -> None` | Bind the WebSocket server and start accepting clients. Idempotent. |
| `stop` | `() -> None` | Close the active client and shut the server down. |
| `capture` | `() -> np.ndarray \| None` | Grab the latest decoded frame as a BGR `numpy` array (OpenCV layout). `None` if no client / no frame yet. |
| `stream` | `() -> Iterator[np.ndarray]` | Generator yielding frames as they arrive. |
| `record` | `(duration: float) -> np.ndarray` | Capture `duration` seconds of frames into a single array. |
| `record_avi` | `(duration: float) -> np.ndarray` | Same, written to an AVI byte buffer. |
| `on_status_changed` | `(cb: Callable[[str, dict], None] \| None)` | Lifecycle events. `status ∈ {"disconnected","connected","streaming","paused"}`. `data` includes `client_address`, `client_name`. Pass `None` to unregister. |
| `is_started` | `() -> bool` | Whether the server is running. |
| `status` | property → `str` | Current lifecycle status. |
| `name` | property → `str` | `"WebSocketCamera"` (display label). |
| `protocol` | property → `"ws" \| "wss"` | Wire protocol (depends on `use_tls`). |
| `ip` / `port` | property | Use to render pairing info / QR codes. |
| `secret` / `encrypt` | property | Inspect security config. |

The class is also a context manager (`with WebSocketCamera(...) as cam: ...`) — `start`/`stop` are called automatically.

## Wire protocol (what clients must send)

- Frames are sent as JPEG / PNG / WebP / BMP / TIFF bytes (or base64 strings — both accepted). Use JPEG for live video.
- Default framing: BPP (Binary Peripheral Protocol). With `secret` set, payloads are HMAC-SHA256 authenticated. With `encrypt=True` they are ChaCha20-Poly1305 encrypted.
- Clients **may** send `?client_name=<name>` URL query param to identify themselves (sanitized; ≤64 chars).
- Clients **may** opt out of BPP by appending `?raw=true` — **only when `secret is None`**, else ignored.

A minimal Python client streaming a webcam at 15 FPS:

```python
import time, base64, cv2
import websockets.sync.client as ws

cap = cv2.VideoCapture(0)
with ws.connect("ws://<board-address>:8080") as sock:
    while True:
        time.sleep(1 / 15)
        ok, frame = cap.read()
        if not ok:
            continue
        _, jpg = cv2.imencode(".jpg", frame)
        sock.send(base64.b64encode(jpg).decode())
```

## Minimal example (phone-camera object detection)

```python
import secrets, string
from datetime import datetime, UTC

from arduino.app_utils import App
from arduino.app_bricks.web_ui import WebUI
from arduino.app_bricks.video_objectdetection import VideoObjectDetection
from arduino.app_peripherals.camera import WebSocketCamera

# 6-digit OTP, shown in the WebUI / QR code for pairing
otp = "".join(secrets.choice(string.digits) for _ in range(6))

ui = WebUI()
camera = WebSocketCamera(secret=otp, encrypt=True)
camera.on_status_changed(lambda evt, data: ui.send_message(evt, data))

detection = VideoObjectDetection(camera, confidence=0.5, debounce_sec=0.0)

ui.on_connect(lambda sid: ui.send_message("welcome", {
    "client_name": camera.name, "secret": otp, "status": camera.status,
    "protocol": camera.protocol, "ip": camera.ip, "port": camera.port,
}))
ui.on_message("override_th", lambda sid, th: detection.override_threshold(th))

def push(detections: dict):
    for label, hits in detections.items():
        for h in hits:
            ui.send_message("detection", {
                "content": label,
                "confidence": h.get("confidence"),
                "timestamp": datetime.now(UTC).isoformat(),
            })

detection.on_detect_all(push)

App.run()
```

Source: `arduino/app-bricks-examples/examples/mobile-video-generic-object-detection`.

## Gotchas

- Only **one client** at a time. Additional connections are rejected.
- `capture()` returns a **BGR** `numpy` array (OpenCV convention), not RGB.
- `encrypt=True` without a `secret` raises `RuntimeError`.
- `use_tls=True` + `encrypt=True` → `encrypt` is force-disabled (TLS already encrypts).
- `camera.ip` reflects the `HOST_IP` env var when set by App Lab; otherwise it is the bind address `"0.0.0.0"`. For QR codes prefer the value the *browser* sees (`window.location.hostname`) and use `camera.port` only.
- When passed into a video brick (e.g. `VideoObjectDetection(camera, ...)`), do **not** also call `camera.start()` yourself — the brick owns the lifecycle.
- If you want a direct embedded preview, mount the `/embed` endpoint exposed by the consuming brick (e.g. `video_objectdetection`) rather than rendering frames manually.
- Pair OTP / secret must be shared with the client out-of-band — typically by rendering it (plus a QR) in the WebUI.
