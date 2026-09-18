---
name: web-console-cdp-upload
description: 平台只有网页后台、没有上传 API 时，用本机已装的 Chrome/Edge + CDP 自动化走完上传/发布/填表流程——复用真实登录态、零安装（不下载 500MB Chromium）、能给文件选择框塞文件。当用户说「只能网页上传」「后台发布」「网页填表自动提交」「发布到技能市场/各平台控制台」「复用已登录的浏览器」时使用。
version: 1.0.0
category: 开发编程
platforms: [WorkBuddy, Claude Code, Codex]
license: MIT
displayName:
  zh: 网页后台上传自动化（免安装浏览器）
  en: Web Console Upload via CDP (No Install)
agent_created: true
---

# 网页后台上传 / 发布 / 填表自动化

## 这个技能解决什么

有些平台**只给网页入口**：没有 CLI、没有可用的 API（或 API 没 token 直接 401），
还必须带着登录态才能走完。典型如 SkillHub 发布页、各类云控制台、内容后台的上传表单。

本技能的做法：**不装任何浏览器**，直接遥控用户机器上已有的 Chrome / Edge
（用 Chrome DevTools Protocol）。对比 `agent-browser` / Playwright 方案：

| | 本技能 | agent-browser / Playwright |
|---|---|---|
| 下载体积 | 0（只依赖 `websocket-client`，几十 KB） | Chromium 约 500MB |
| 登录态 | **复用用户日常浏览器的登录态** | 全新 profile，通常要重新登录 |
| 适用 | 有头、需登录的后台操作 | 无头采集、批量自动化 |

依赖：`pip install websocket-client`（多数环境已具备）。

## 流程

### 1. 探测浏览器

```bash
python scripts/cdp.py chrome-path
```
按 Windows / macOS / Linux 依次探测 Chrome、Edge、Chromium，打印可执行文件路径；找不到输出 `NOT_FOUND`。

### 2. 带调试端口启动 —— 必须用「常驻后台任务」

**这一步最容易卡住：浏览器会随启动它的 shell 一起被回收。**

| 启动方式 | 结果 |
|---|---|
| 普通 shell 调用 + `&` | ❌ shell 退出即死，调试端口立刻不通 |
| `cmd //c start`（Git Bash） | ❌ MSYS 路径转换会把 `C:\Program Files\...` 拆坏，报 `'hrome' not recognized` |
| `Start-Process`（PowerShell） | ❌ 进程起来了但端口不监听 |
| **后台常驻任务**（Bash 工具 `run_in_background=true`） | ✅ 稳定常驻 |

```bash
"<上一步探测到的路径>" --remote-debugging-port=9222 \
  --user-data-dir="<临时目录，建议新建，别污染日常 profile>" \
  --no-first-run --no-default-browser-check "https://目标页"
```

想长期省事，可在启动参数里加 `--remote-allow-origins=*`，这样第 3 步就不必抑制 Origin。

### 3. 连通性自检（两条都要做）

```bash
unset HTTP_PROXY HTTPS_PROXY http_proxy https_proxy ALL_PROXY all_proxy   # 本机代理会拦 localhost
curl -s --noproxy '*' http://127.0.0.1:9222/json/version                  # 能返回即通
```

连 WebSocket 时（脚本已处理）：Chrome 新版校验握手来源，需
`websocket.create_connection(ws_url, suppress_origin=True)`，否则 `Handshake status 403 Forbidden`。

### 4. 读页面 → 判断状态

```bash
python scripts/cdp.py eval "document.body.innerText.slice(0,1500)"
```
用正文判断「是否已登录 / 列表里有没有这条 / 当前什么状态」，比截图快得多。
需要换页面时用 `CDP_MATCH` 环境变量指定 URL 片段；留空则取第一个页面标签。

### 5. 点击

先按 `textContent.trim()` 精确匹配元素再 `el.click()`：

```js
var spans=[].slice.call(document.querySelectorAll('span,button,a'));
for (var i=0;i<spans.length;i++){
  var s=spans[i];
  if((s.textContent||'').trim()!=='更新') continue;
  var box=s, hit=null;
  for(var k=0;k<5 && box;k++){ box=box.parentElement;
    if(box && (box.textContent||'').indexOf('slug: 唯一标识')>=0){ hit=box; break; } }
  if(hit){ s.click(); return 'clicked'; }
}
```
⚠️ 别用 `closest('div')` 直接找卡片——按钮常在更外层的兄弟容器里，
要**向上逐层找包含唯一标识文本的那一层**再定位。

### 6. 填表（React / Vue 受控组件）

直接改 `.value` 不生效，必须走原生 setter：

```js
var proto = el.tagName === 'TEXTAREA' ? HTMLTextAreaElement.prototype : HTMLInputElement.prototype;
Object.getOwnPropertyDescriptor(proto, 'value').set.call(el, v);
el.dispatchEvent(new Event('input', {bubbles: true}));
```

⚠️ **但带异步后端校验的字段（如「slug 是否可用」）光这样还不够**：
合成事件常常不触发它的防抖校验，界面会一直挂着上一次的报错（如 `Slug 不可用`、
输入框 `aria-invalid="true"`），改了值也不重算，于是提交必然失败。
必须用**真实输入事件**再补一次 blur：

```python
c.call('DOM.focus', nodeId=nid)
c.eval("var e=document.querySelector('#skill-slug'); e.focus(); e.setSelectionRange(0,e.value.length);")
c.call('Input.insertText', text='目标值')
c.eval("var e=document.querySelector('#skill-slug'); e.blur();")
# 等 5~6 秒后检查 aria-invalid 是否为 false，再提交
```

判据：提交前确认目标输入框的 `aria-invalid` 是 `false`、页面里没有残留报错文本。
「改完值但报错不消失」≠ 值真的不可用——先怀疑校验没被触发（真被占用时
重新输入后仍会报，可据此区分）。

### 7. 上传文件

```bash
python scripts/cdp.py file 'input[accept=".zip,application/zip"]' /abs/path/file.zip
```
然后补发一次 change 事件让前端解析：

```js
inp.dispatchEvent(new Event('change', {bubbles:true}));
```
解析成功的标志是界面出现「已选择 N 个文件，总大小 X KB」——**以这条为准**。

### 8. 提交与复核

点提交按钮后对话框通常自动关闭。复核要 `location.reload()` 再读列表状态
（如「安全审核中 / V 1.0.1」）。最后连 `/json/version` 的 `webSocketDebuggerUrl`
发 `Browser.close` 收尾，别留僵尸浏览器。

## 必坑清单

1. **常驻后台启动**，否则端口永远不通（见第 2 步）。
2. **`suppress_origin=True`**，否则 WS 403。
3. **localhost 也要 bypass 代理**，否则 502。
4. **隐藏 file input 常不止一个**：文件夹那个带 `webkitdirectory`，压缩包那个带
   `accept=".zip,application/zip"`——**按 accept 挑，别按序号猜**。
5. **弹窗会重渲染**：注入文件后原 input 常被移除，`querySelector` 返回 null 不算失败，看界面文本确认。
6. **提交后对话框关闭通常即成功**；不确定就刷新页面看状态字段。
7. 若遇到站点有反爬/WAF，优先复用用户真实 profile（登录态 + 指纹），这是本方案相对无头浏览器的最大优势。

## 脚本接口

| 子命令 | 作用 |
|---|---|
| `chrome-path` | 探测本机浏览器可执行文件 |
| `eval "<js>"` | 页面上下文求值 |
| `shot <out.png>` | 截图 |
| `file <选择器> <路径>` | 给 `<input type=file>` 塞文件 |

环境变量：`CDP_PORT`（默认 9222）、`CDP_MATCH`（默认取第一个页面标签）。
