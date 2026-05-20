# mitmproxy Guide

## 1. What is mitmproxy

mitmproxy is a man-in-the-middle proxy tool that intercepts, inspects, and modifies HTTP/HTTPS requests and responses.

## 2. Three Tools Compared

| Tool | Interface | Use Case |
|------|-----------|----------|
| `mitmproxy` | Terminal interactive TUI | Real-time inspection, editing, intercepting requests |
| `mitmweb` | Browser Web UI (default `http://127.0.0.1:8081`) | Visual inspection, similar to Chrome DevTools |
| `mitmdump` | No UI, plain text output | Scripting, automation, logging |

All three share the same proxy engine — only the presentation differs. Command parameters are interchangeable.

## 3. Forward Proxy vs Reverse Proxy

### Core Difference

- **Forward proxy** — sits on the client side, makes requests on behalf of the client. The client knows the proxy exists (must be configured).
- **Reverse proxy** — sits on the server side, accepts requests on behalf of the server. The client is unaware, thinking it's talking directly to the real service.

```
Forward:  Client → [Proxy] → Server
          Client knows the proxy, server doesn't

Reverse:  Client → [Proxy] → Server
          Server knows the proxy, client doesn't
```

**"Forward/Reverse" refers to who the proxy works for:** for the client = forward, for the server = reverse.

### When to Use Which

| | Reverse Proxy | Forward Proxy |
|---|---|---|
| Server | Must change port | No change |
| Client | No change | Must configure proxy |
| Best for | Many/uncontrollable clients | Uncontrollable server |

## 4. Reverse Proxy Usage

Suppose the original service runs on port 23333. Move it to 23334 and let mitmproxy take over 23333:

```bash
# Pick any of the three tools
mitmproxy --mode reverse:http://localhost:23334 --listen-port 23333
mitmweb   --mode reverse:http://localhost:23334 --listen-port 23333
mitmdump  --mode reverse:http://localhost:23334 --listen-port 23333
```

Clients don't change — they still access 23333. mitmproxy transparently records and forwards to 23334.

## 5. Forward Proxy Usage

Server stays unchanged. Clients send requests through the proxy:

```bash
# Pick any of the three tools
mitmproxy --listen-port 8080
mitmweb   --listen-port 8080
mitmdump  --listen-port 8080
```

### Client Proxy Configuration

#### PowerShell (environment variables)

```powershell
$env:http_proxy = "http://localhost:8080"
$env:https_proxy = "http://localhost:8080"
$env:no_proxy = ""
python your_script.py
```

> **Important:** `$env:no_proxy = ""` must be set, otherwise Python's `requests`/`httpx` will bypass the proxy for localhost traffic.

#### CMD

```cmd
set http_proxy=http://localhost:8080
set https_proxy=http://localhost:8080
set no_proxy=
python your_script.py
```

> **Note:** In PowerShell, `set` is an alias for `Set-Variable`, which sets a PowerShell variable, not an environment variable — Python won't see it. You must use the `$env:` syntax.

#### Restoring Environment Variables

Close the terminal window (variables only exist in the current session), or clear them manually:

```powershell
# PowerShell
$env:http_proxy = ""
$env:https_proxy = ""
$env:no_proxy = ""
```

## 6. How Forward Proxy Works

Client code still targets `localhost:23333`, but with `http_proxy` set, the HTTP library connects to the proxy (8080) instead, including the full URL in the request line:

```
GET http://localhost:23333/api HTTP/1.1    ← full URL, proxy uses this to forward
Host: localhost:23333
```

The proxy reads the full URL from the request line to know where to forward, fetches the response, then returns it to the client.

## 7. mitmweb Web Interface

Running `mitmweb` automatically opens a browser UI:

- **Proxy port** (e.g., 8080) — handles traffic
- **Web UI port** (default 8081) — view captured traffic at `http://127.0.0.1:8081`

The Web UI port is customizable:

```bash
mitmweb --listen-port 8080 --web-port 9090
```

## 8. Transparent Proxy on Windows

Windows lacks iptables, so transparent proxying (no changes to client or server) isn't practical like on Linux. The realistic options are forward or reverse proxy — pick whichever side is cheaper to modify.

## 9. Installation Notes

```bash
pip install mitmproxy
```

mitmproxy may install an incompatible version of `typing_extensions`, causing errors in other libraries (e.g., `anthropic`):

```
ImportError: cannot import name 'List' from 'typing_extensions'
```

Fix:

```bash
pip install --upgrade typing_extensions
```
<img width="2276" height="1284" alt="image" src="https://github.com/user-attachments/assets/ec2c7454-6bad-4cdc-ab92-83b7e9277099" />
