# Glimpse

Native macOS micro-UI for scripts and agents.

Glimpse opens a native WKWebView window in under 50ms and speaks a bidirectional JSON Lines protocol over stdin/stdout. No Electron, no browser, no runtime dependencies — just a tiny Swift binary and a Node.js wrapper.

## Requirements

- macOS (any recent version)
- Xcode Command Line Tools: `xcode-select --install`
- Node.js 18+

## Install

```bash
npm install glimpse
```

`npm install` automatically compiles the Swift binary via a `postinstall` hook (~2 seconds). See [Compile on Install](#compile-on-install) for details.

**Manual build:**
```bash
npm run build
# or directly:
swiftc src/glimpse.swift -o src/glimpse
```

## Quick Start

```js
import { open } from 'glimpse';

const win = open(`
  <html>
    <body style="font-family:sans-serif; padding:2rem;">
      <h2>Hello from Glimpse</h2>
      <button onclick="glimpse.send({ action: 'greet' })">Say hello</button>
    </body>
  </html>
`, { width: 400, height: 300, title: 'My App' });

win.on('message', (data) => {
  console.log('Received:', data); // { action: 'greet' }
  win.close();
});

win.on('closed', () => process.exit(0));
```

## API Reference

### `open(html, options?)`

Opens a native window and returns a `GlimpseWindow`. The HTML is displayed once the WebView signals ready.

```js
import { open } from 'glimpse';

const win = open('<html>...</html>', {
  width:  800,    // default: 800
  height: 600,    // default: 600
  title:  'App',  // default: "Glimpse"
});
```

### GlimpseWindow

`GlimpseWindow` extends `EventEmitter`.

#### Events

| Event | Payload | Description |
|-------|---------|-------------|
| `ready` | — | WebView is loaded and ready to receive commands |
| `message` | `data: object` | Message sent from the page via `window.glimpse.send(data)` |
| `closed` | — | Window was closed (by user or via `.close()`) |
| `error` | `Error` | Process error or malformed protocol line |

```js
win.on('ready',   ()    => console.log('window ready'));
win.on('message', (msg) => console.log('from page:', msg));
win.on('closed',  ()    => process.exit(0));
win.on('error',   (err) => console.error(err));
```

#### Methods

**`win.send(js)`** — Evaluate JavaScript in the WebView.
```js
win.send(`document.body.style.background = 'coral'`);
win.send(`document.getElementById('status').textContent = 'Done'`);
```

**`win.setHTML(html)`** — Replace the entire page content.
```js
win.setHTML('<html><body><h1>Step 2</h1></body></html>');
```

**`win.close()`** — Close the window programmatically.
```js
win.close();
```

### JavaScript Bridge (in-page)

Every page loaded by Glimpse gets a `window.glimpse` object injected at document start:

```js
// Send any JSON-serializable value to Node.js → triggers 'message' event
window.glimpse.send({ action: 'submit', value: 42 });

// Close the window from inside the page
window.glimpse.close();
```

## Protocol

Glimpse uses a newline-delimited JSON (JSON Lines) protocol. Each line is a complete JSON object. This makes it easy to drive the binary from any language.

### Stdin → Glimpse (commands)

**Set HTML** — Replace page content. HTML must be base64-encoded.
```json
{"type":"html","html":"<base64-encoded HTML>"}
```

**Eval JavaScript** — Run JS in the WebView.
```json
{"type":"eval","js":"document.title = 'Updated'"}
```

**Close** — Close the window and exit.
```json
{"type":"close"}
```

### Stdout → Host (events)

**Ready** — WebView finished loading initial blank page. Send HTML after this.
```json
{"type":"ready"}
```

**Message** — Data sent from the page via `window.glimpse.send(...)`.
```json
{"type":"message","data":{"action":"submit","value":42}}
```

**Closed** — Window closed (by user or via close command).
```json
{"type":"closed"}
```

Diagnostic logs are written to **stderr** (prefixed `[glimpse]`) and do not affect the protocol.

## CLI Usage

Drive the binary directly from any language — shell, Python, Ruby, etc.

```bash
# Basic usage
echo '{"type":"html","html":"PGh0bWw+PGJvZHk+SGVsbG8hPC9ib2R5PjwvaHRtbD4="}' \
  | ./src/glimpse --width 400 --height 300 --title "Hello"
```

Available flags:

| Flag | Default | Description |
|------|---------|-------------|
| `--width N` | `800` | Window width in pixels |
| `--height N` | `600` | Window height in pixels |
| `--title STR` | `"Glimpse"` | Window title bar text |

**Shell example — encode HTML and pipe it in:**
```bash
HTML=$(echo '<html><body><h1>Hi</h1></body></html>' | base64)
{
  echo "{\"type\":\"html\",\"html\":\"$HTML\"}"
  cat  # keep stdin open so the window stays up
} | ./src/glimpse --width 600 --height 400
```

**Python example:**
```python
import subprocess, base64, json

html = b"<html><body><h1>Hello from Python</h1></body></html>"
proc = subprocess.Popen(
    ["./src/glimpse", "--width", "500", "--height", "400"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE
)

cmd = json.dumps({"type": "html", "html": base64.b64encode(html).decode()})
proc.stdin.write((cmd + "\n").encode())
proc.stdin.flush()

for line in proc.stdout:
    msg = json.loads(line)
    if msg["type"] == "ready":
        print("Window is ready")
    elif msg["type"] == "message":
        print("From page:", msg["data"])
    elif msg["type"] == "closed":
        break
```

## Compile on Install

Every Mac ships with `swiftc` once Xcode Command Line Tools are installed — no Xcode IDE required. Glimpse takes advantage of this: running `npm install` triggers a `postinstall` script that compiles `src/glimpse.swift` into a native binary in about 2 seconds.

```
> glimpse@0.1.0 postinstall
> npm run build

swiftc src/glimpse.swift -o src/glimpse  ✓
```

**If compilation fails**, the most common cause is missing Xcode CLT:
```bash
xcode-select --install
```

To recompile manually at any time:
```bash
npm run build
```

## License

MIT
