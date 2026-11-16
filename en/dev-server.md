# Dev Server & Live Reload

Luany ships two development servers: `luany serve`, a plain PHP built-in server, and `luany dev`, the Luany Dev Engine (LDE), which adds a file watcher and browser live reload. This page covers both, their requirements and how live reload works.

## luany serve

```bash
luany serve                  # http://localhost:8000
luany serve 127.0.0.1 8080   # custom host and port
```

`serve` runs PHP's built-in server with `public/` as document root. Arguments are positional:

| Argument | Default | Accepts |
|---|---|---|
| `host` | `localhost` | Host name, IPv4 address or bracketed IPv6 (`[::1]`) |
| `port` | `8000` | Integer 1–65535 |

Stop it with `Ctrl+C`. Use `serve` when you do not need live reload or do not have Node.js installed.

## luany dev

```bash
luany dev
```

```bash
  ┌─────────────────────────────────────┐
  │   🚀  Luany Dev Engine  (LDE v1)    │
  └─────────────────────────────────────┘

  ✓  PHP server    →  http://localhost:8000
  ✓  Live reload   →  ws://localhost:35729
  ✓  CSS inject    →  active

  Press Ctrl+C to stop.
```

`luany dev` starts two child processes and supervises them:

1. **PHP built-in server** on `host:port`, serving `public/`.
2. **Node.js watcher** (`watcher.js`, shipped with the CLI) that watches project files and runs a WebSocket server.

The browser talks to PHP directly — there is no proxy — and receives reload messages over the WebSocket.

| Argument | Default | Description |
|---|---|---|
| `host` | `localhost` | PHP server host |
| `port` | `8000` | PHP server port |
| `ws-port` | `35729` | WebSocket port for live reload |

```bash
luany dev localhost 8080          # custom HTTP port
luany dev localhost 8000 35730    # custom WebSocket port
```

The HTTP and WebSocket ports must differ. Before starting, `dev` checks that both ports are free; if one is taken it stops with a suggestion such as `luany dev localhost 8080`.

### Requirements

| Requirement | Why |
|---|---|
| Node.js on `PATH` (`node` or `nodejs`) | Runs the watcher |
| `npm install` in the project | Installs `chokidar` and `ws`, declared in the skeleton's `package.json` |
| `APP_ENV=development` (or `local`) | Enables `LiveReloadMiddleware` |
| `LiveReloadMiddleware` in the HTTP Kernel | Injects the browser client (registered in the skeleton) |

```bash
npm install
luany dev
```

If Node.js or the packages are missing, `dev` stops with a clear message (`Required Node.js packages not installed: chokidar, ws — Run: npm install`). The skeleton's `package.json` also maps `npm run dev` to `luany dev`.

If `APP_ENV` is neither `development` nor `local`, `dev` prints a warning but still starts. `APP_ENV` is read from the real environment first, then from `.env`.

### What is watched

| Path | Change | Browser action |
|---|---|---|
| `views/**/*.lte` | change, add | Full reload |
| `app/**/*.php`, `routes/**/*.php`, `config/**/*.php` | change, add | Full reload |
| `public/assets/**/*.css` | change, add | **CSS injection** (no reload) |
| `public/assets/**/*` (other files) | change, add | Full reload |
| Any watched file | delete | Full reload |

Hidden files and folders, `node_modules`, the top-level `vendor/` and `storage/cache` / `storage/logs` are ignored. Events are debounced (40 ms): a burst of saves produces one action, a full reload wins over CSS injection, and several changed stylesheets are all injected.

**CSS injection** updates every `<link rel="stylesheet">` whose file name matches the changed file, adding a `?lde=<timestamp>` cache-buster. If no link matches, the page reloads.

The terminal shows each event:

```bash
[LDE] Changed: public/assets/css/app.css
[LDE] inject-css → app.css (1 client)
[LDE] Changed: views/pages/home.lte
[LDE] reload (1 client)
```

## LiveReloadMiddleware

`Luany\Framework\Http\Middleware\LiveReloadMiddleware` connects the browser to the watcher. The skeleton registers it first in `app/Http/Kernel.php`, so it sees the final response:

```php
protected array $middleware = [
    LiveReloadMiddleware::class, // `luany dev` live reload — development + localhost only
    CsrfMiddleware::class,
];
```

The middleware does nothing unless **both** conditions hold:

- `APP_ENV` is `development` or `local`;
- the request comes from the loopback interface (`REMOTE_ADDR` is `::1` or starts with `127.`).

When active, it:

1. Serves the browser client at `/__luany_dev/client.js` (from `LDE_CLIENT_PATH`, set by `luany dev`, or `vendor/luany/cli` as a fallback).
2. Injects `<script>window.__LDE_WS_PORT__ = 35729;</script>` and the client `<script>` before the last `</body>` of the response.

Only successful (2xx) responses are touched, and only when the `Content-Type` is HTML or not set (views rendered with `view()` have none). JSON, files, redirects and pages without `</body>` are left alone.

The client reconnects automatically with exponential back-off (up to 5 seconds), so an open page keeps receiving changes after you restart `luany dev`.

> **Tip** Keep `LiveReloadMiddleware` in the Kernel in production: with `APP_ENV=production` it returns immediately. See [Deployment](deployment.md).

## Security

- The WebSocket server listens on `127.0.0.1` only.
- It accepts connections only from pages served by `localhost`, `127.0.0.1`, `::1`, `*.localhost` or the host given to `luany dev`; other origins are rejected and logged.
- The middleware never injects anything for non-loopback requests or outside development.

Because both the WebSocket and the client use the loopback address, live reload works in a browser on the same machine. Other devices on your network can still open the site (with `luany dev 0.0.0.0`), but without live reload.

## Stopping and failures

Press `Ctrl+C` to stop both processes. If either child exits unexpectedly — for example the PHP server cannot bind its port — LDE stops the other one and exits with the child's exit code (or `1`), so scripts can detect the failure.

## Windows notes

- Node.js is located with `where`; make sure `node` is on the `PATH` of the terminal you use.
- `Ctrl+C` and `Ctrl+Break` are handled natively; child processes are stopped with `taskkill /F /T`.
- Paths with spaces are supported (processes are started without a shell).
- If your PHP `variables_order` omits `E` (common in WAMP setups), the watcher still inherits the full environment.
