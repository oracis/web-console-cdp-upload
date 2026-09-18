# web-console-cdp-upload

Drive the Chrome/Edge you already have installed — via the **Chrome DevTools Protocol** — to finish
upload / publish / form-submit tasks on websites that only expose a web console.

No browser download. No Playwright. No Selenium. Just the browser that is already on your machine,
already logged in.

> 中文说明见下方 [中文](#中文说明)。

## Why

Some platforms give you **nothing but a web page**: no CLI, no usable API (or the API returns 401
without a token), and the flow only completes if you are logged in. Typical cases: publishing a skill
to a marketplace, uploading an artifact to a cloud console, filling a long review form.

The usual answer — `agent-browser`, Playwright, Puppeteer — means downloading a ~500 MB Chromium and
logging in again from scratch in a fresh profile. This repo takes the opposite route:

| | This repo | agent-browser / Playwright |
|---|---|---|
| Download size | 0 (needs `websocket-client`, a few KB) | ~500 MB Chromium |
| Login state | **Reuses your real browser session** | Fresh profile, usually needs re-login |
| Best for | Headful, login-required console work | Headless scraping, batch automation |

## Install

```bash
pip install websocket-client
```

Works on Windows / macOS / Linux. Python 3.8+.

## Quick start

```bash
# 1. Find the browser on this machine
python scripts/cdp.py chrome-path

# 2. Launch it with a debugging port. IMPORTANT: run this as a persistent background task,
#    otherwise the browser dies as soon as the shell exits.
"/path/from/step/1" --remote-debugging-port=9222 \
  --user-data-dir="/tmp/cdp-profile" \
  --no-first-run --no-default-browser-check "https://target.example/dashboard"

# 3. Sanity check (also: bypass any local HTTP proxy before talking to localhost)
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy ALL_PROXY all_proxy
curl -s --noproxy '*' http://127.0.0.1:9222/json/version

# 4. Read the page
python scripts/cdp.py eval "document.body.innerText.slice(0,1500)"

# 5. Upload a file into a hidden <input type=file>
python scripts/cdp.py file 'input[accept=".zip,application/zip"]' /abs/path/file.zip
```

## CLI

| Command | What it does |
|---|---|
| `chrome-path` | Detect Chrome / Edge / Chromium on this machine |
| `eval "<js>"` | Evaluate JS in the page context |
| `shot <out.png>` | Screenshot |
| `file <selector> <path>` | Inject a file into `<input type=file>` via `DOM.setFileInputFiles` |

Environment variables: `CDP_PORT` (default `9222`), `CDP_MATCH` (URL fragment; empty = first tab).

## Pitfalls (all of these cost real debugging time)

1. **Launch in a persistent background task.** Plain shell `&` kills the browser when the shell exits
   and the debug port stops answering. `cmd //c start` gets mangled by Git Bash's MSYS path
   conversion (`'hrome' is not recognized`). PowerShell `Start-Process` starts the process but the
   port never listens.
2. **Suppress the WebSocket Origin.** Chrome validates the WS handshake origin since v153:
   `websocket.create_connection(ws_url, suppress_origin=True)`, or you get
   `Handshake status 403 Forbidden`. (Already handled in `scripts/cdp.py`.)
3. **Bypass your local proxy for localhost too.** Otherwise `127.0.0.1:9222` returns 502.
4. **Several hidden file inputs can exist.** The folder one has `webkitdirectory`, the archive one has
   `accept=".zip,application/zip"` — select by `accept`, never by index.
5. **Dialogs re-render.** After file injection the original input is often removed; `querySelector`
   returning `null` is not a failure. Check the visible text ("3 files selected, 11.9 KB").
6. **React-controlled inputs need the native setter:**
   ```js
   var proto = el.tagName === 'TEXTAREA' ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
   Object.getOwnPropertyDescriptor(proto, 'value').set.call(el, v);
   el.dispatchEvent(new Event('input', {bubbles: true}));
   ```
7. **Fields with async server-side validation (e.g. "is this slug available?") do not re-validate on
   synthetic events.** The previous error (`Slug unavailable`, `aria-invalid="true"`) stays on screen
   and every submit fails. Use real input + blur:
   ```python
   c.call('DOM.focus', nodeId=nid)
   c.eval("var e=document.querySelector('#skill-slug'); e.focus(); e.setSelectionRange(0,e.value.length);")
   c.call('Input.insertText', text='my-slug')
   c.eval("var e=document.querySelector('#skill-slug'); e.blur();")
   ```
   Wait 5–6 s, confirm `aria-invalid === "false"`, then submit.
8. **Close the browser when done:** connect to the `webSocketDebuggerUrl` from `/json/version` and
   send `Browser.close`.

## Repo layout

```
SKILL.md          # skill definition (agent-facing, Chinese)
manifest.yaml     # metadata for skill marketplaces
scripts/cdp.py    # the CDP client
```

## 中文说明

有些平台**只有网页入口**：没有 CLI、没有可用 API（或 API 无 token 直接 401），且必须带登录态才能走完。
本技能不装任何浏览器，直接通过 CDP 遥控本机已有的 Chrome / Edge：

- **零安装**：只依赖 `websocket-client`，省下 `agent-browser` / Playwright 那 500MB 的 Chromium
- **复用登录态**：用的是你日常浏览器的会话，不用重新扫码登录
- **能给文件选择框塞文件**：`DOM.setFileInputFiles` 直接注入，绕过系统文件对话框

用法见上方 Quick start；踩坑清单就是上面那 8 条（每一条都是实际调通过程中撞出来的）。

## License

MIT
