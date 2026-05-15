# Frontend — `assets/` (HTML / CSS / JS)

The `WebUI` brick serves `assets/` as the static root on port 7000. The page must be self-contained: **no CDN at runtime**. All third-party JS/CSS lives under `assets/js/vendor/` (or `assets/css/vendor/`), downloaded once and committed.

## Directory layout

```
assets/
  index.html          # entrypoint, required by the brick
  css/
    app.css           # your styles
    vendor/           # optional, third-party CSS
  js/
    app.js            # your code (ES modules OK, no bundler)
    vendor/
      socket.io.min.js
      VERSIONS.md     # manifest of vendored libs
```

## Vendoring third-party JS/CSS

Procedure when you need a new library:

1. Find the official minified URL (jsDelivr or the vendor's CDN).
2. Download it into `assets/js/vendor/` (or `assets/css/vendor/`) with `curl`:
   ```bash
   curl -L -o assets/js/vendor/socket.io.min.js \
     https://cdn.socket.io/4.7.5/socket.io.min.js
   ```
3. Append an entry to `assets/js/vendor/VERSIONS.md`:
   ```markdown
   | File | Version | Source |
   | --- | --- | --- |
   | socket.io.min.js | 4.7.5 | https://cdn.socket.io/4.7.5/socket.io.min.js |
   ```
4. Reference with a relative path in `index.html`:
   ```html
   <script src="js/vendor/socket.io.min.js"></script>
   ```

**Never** put a CDN URL in a `<script src>` or `<link href>` at runtime.

## Socket.IO client

The WebUI brick mounts Socket.IO at `/socket.io`. The default client (loaded as a global `io`) auto-connects to the same origin:

```html
<script src="js/vendor/socket.io.min.js"></script>
<script type="module" src="js/app.js"></script>
```

```js
// assets/js/app.js
const socket = io();

socket.on("connect", () => console.log("connected", socket.id));
socket.on("telemetry", (msg) => updateChart(msg.value));

// send a message handled by ui.on_message("command", ...) on the Python side
function sendCommand(value) {
  socket.emit("command", { value });
}

// for replies emitted by the server as "<event>_response"
socket.on("command_response", (data) => console.log("ack", data));
```

REST endpoints exposed via `ui.expose_api(...)` are reachable with `fetch`:

```js
const status = await fetch("/api/status").then(r => r.json());
```

Pin the Socket.IO client version to match the brick (server is `python-socketio`; client `4.x` is compatible).

## HTML skeleton

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>App</title>
    <link rel="stylesheet" href="css/app.css" />
  </head>
  <body>
    <header class="topbar">
      <h1>App title</h1>
      <span id="status" class="pill">offline</span>
    </header>

    <main class="grid">
      <section class="card">
        <h2>Telemetry</h2>
        <p id="value" class="metric">–</p>
      </section>
      <section class="card">
        <h2>Controls</h2>
        <button id="btn-toggle" class="btn-primary">Toggle</button>
      </section>
    </main>

    <script src="js/vendor/socket.io.min.js"></script>
    <script type="module" src="js/app.js"></script>
  </body>
</html>
```

## Visual style (default)

Use a small, opinionated baseline. CSS variables, dark theme by default, system font stack, Arduino-leaning palette.

```css
/* assets/css/app.css */
:root {
  --bg: #0e1116;
  --surface: #161b22;
  --border: #243041;
  --text: #e6edf3;
  --muted: #8b949e;
  --accent: #008184;        /* Arduino teal */
  --accent-hi: #00b3b8;
  --danger: #ff6b6b;
  --radius: 12px;
  --shadow: 0 4px 14px rgba(0, 0, 0, 0.35);
}

* { box-sizing: border-box; }

body {
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  background: var(--bg);
  color: var(--text);
}

.topbar {
  display: flex; align-items: center; justify-content: space-between;
  padding: 1rem 1.5rem;
  border-bottom: 1px solid var(--border);
}

.pill {
  padding: 0.25rem 0.75rem; border-radius: 999px;
  background: var(--surface); color: var(--muted);
  font-size: 0.85rem;
}
.pill.online { color: var(--accent-hi); }

.grid {
  display: grid; gap: 1rem; padding: 1.5rem;
  grid-template-columns: repeat(auto-fit, minmax(260px, 1fr));
}

.card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 1.25rem;
  box-shadow: var(--shadow);
}

.metric { font-size: 2.5rem; font-weight: 600; margin: 0.5rem 0 0; }

.btn-primary {
  background: var(--accent); color: #fff;
  border: 0; border-radius: var(--radius);
  padding: 0.6rem 1.1rem; font-weight: 600; cursor: pointer;
  transition: background 0.15s ease;
}
.btn-primary:hover { background: var(--accent-hi); }
```

## Rules

1. No CDN `<script src>` / `<link href>` at runtime — vendor everything under `assets/.../vendor/`.
2. `index.html` must exist in `assets/` or the WebUI brick fails to start.
3. Use ES modules (`type="module"`) — no bundler, no transpiler.
4. Keep `app.js` small; split into multiple modules under `assets/js/` if it grows.
5. Use `fetch("/api/...")` for REST, Socket.IO for realtime. Do not poll a REST endpoint at high frequency.
