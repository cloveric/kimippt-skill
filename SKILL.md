---
name: kimippt
description: "Use this skill only when the user explicitly writes `kimippt`, `/kimippt`, `帮我生成kimippt`, `kimi ppt`, `kimi slides`, or asks to generate a PPT via Kimi / kimi.com/slides. Drives https://www.kimi.com/slides in the user's EXISTING Chrome via `agent-browser` (CDP port 9222). Uploads the user's document, picks 智能布局 + 自由风格 (unless the user names a different style), waits patiently for generation, downloads the .pptx, and delivers it. Do NOT trigger for generic PPT requests — use the `pptx` or `minimaxppt` skills for those."
metadata:
  short-description: Generate a PPT by driving kimi.com/slides in the user's logged-in Chrome via agent-browser (CDP 9222)
  trigger-aliases:
    - kimippt
    - /kimippt
    - 帮我生成kimippt
    - 用kimi生成ppt
    - kimi ppt
    - kimi slides
allowed-tools: Bash(agent-browser:*)
---

# Kimi PPT Skill

Drives [https://www.kimi.com/slides](https://www.kimi.com/slides) in the user's **already-running, already-logged-in Chrome** to generate a PPT from a document the user uploaded, then downloads and delivers the result.

## Trigger Gate

Only run when the user explicitly writes one of:

- `kimippt` / `/kimippt`
- `帮我生成kimippt` / `帮我用kimi生成ppt`
- `kimi ppt` / `kimi slides`

For generic "做个 PPT" requests, use `pptx` or `minimaxppt` instead.

## The Cardinal Rule: Reuse Existing Chrome

**Do NOT launch a new browser. Do NOT use the chrome-devtools MCP, Playwright MCP, or any tool that could spawn its own isolated session.** The only approved driver is the `agent-browser` CLI, and it **must** connect to the existing Chrome on `127.0.0.1:9222` before doing anything else. If you skip this step, you will end up in a logged-out ghost session and waste the user's time.

The user keeps Chrome running with `--remote-debugging-port=9222`. That port IS the entry point to their current browser — it is not a fallback, it is the primary mechanism.

## Preflight

Before doing anything Kimi-specific:

1. **Connect to the existing Chrome**
   ```bash
   agent-browser connect 9222
   ```
   If that command errors, surface the error and stop. Tell the user: "没连到 9222 — 你的 Chrome 可能没开 `--remote-debugging-port=9222`，或端口被占用了。" Do **not** retry with anything that launches a new browser.

2. **Verify you are on the real user's Chrome** (not a fresh tab)
   ```bash
   agent-browser get title
   agent-browser get url
   ```
   Report both back to the user in one line ("当前 Chrome 标签是 `<title>` @ `<url>`") before moving on. If title/url look like an empty about:blank session or a brand-new Chrome, something is wrong — stop and flag it.

3. **Resolve the input document path.** Telegram uploads land at `~/.cctb/default/workspace/.telegram-files/<id>/input/<file>`. Use the path from the user's message verbatim (absolute). Do not re-fetch or re-upload.

## Default Settings

Unless overridden in the same message:

- **Layout (布局)** — `智能布局`
- **Category (类别)** — `通用`
- **Style (风格)** — `自由风格`

If the user writes e.g. "用商务风", "换成极简" — pick that style instead.

**Consulting / McKinsey-style requests** (user says "麦肯锡风格", "咨询风格", "BCG 风", "战略报告"):
- Switch **Category** to `商业洞察` first — this swaps the theme list to consulting-flavored options (`翠金洞察 / 灰钢质感 / 赤线锐评 / 藏蓝铜金 / 酒红咨询`).
- Then pick **Theme** `藏蓝铜金` (navy + bronze gold, closest to classic McKinsey palette).
- Page count: ask user or set explicitly via the picker (see Step 4b). Don't leave on `自动页数` for long reports.

## Workflow

### Step 1 — Open kimi.com/slides

```bash
agent-browser open https://www.kimi.com/slides
agent-browser wait --load networkidle
agent-browser snapshot -i
```

### Step 2 — Diagnose login state explicitly

Read the snapshot. Three possible states — **report which one to the user before acting**:

- **(A) Logged in** → you see the slides creation UI (upload affordance, 布局/风格 pickers nearby, user avatar top-right). Proceed to Step 3.
- **(B) Not logged in** → you see `登录` / `扫码登录` / `Login` on a login panel. **Stop.** Tell the user: "当前 Chrome 的 Kimi 未登录，请在浏览器里登录后再 /kimippt。" Do not attempt to automate login.
- **(C) Blocked by verification / captcha / anti-abuse page** → you see a captcha or "异常访问" page. **Stop.** Tell the user what you saw.

Never collapse these three into one "登不了" message.

### Step 3 — Upload the document

**Kimi's quirk**: the `<input type="file">` doesn't exist in DOM until you click the **`+` button** (`.toolkit-trigger-btn`) inside the chat editor.

⚠️ **Identifying the right `+` button**: don't grep snapshot output for "button" — the actual `+` shows up as a **nameless `generic clickable`** (AX role `generic`, not `button`), and it sits **right next to the prompt textbox** in the `.chat-editor-action` area. There's also an unrelated nameless `button` in the sidebar (the "添加联系人" button next to "发起群聊") — clicking that does nothing useful and wastes a turn. Stable identifier:

```bash
agent-browser eval 'document.querySelector(".toolkit-trigger-btn") ? "found" : "missing"'   # always-correct hook
```

After the menu opens, the AX tree exposes a `LabelText "文件和图片" [ref=eN]` — that label **wraps** the real file input but is **not** the input itself, so `upload @eN` will fail with `CDP error: Node is not a file input element`.

Correct flow:

```bash
# Click the + button via its stable class (skips the AX-tree confusion above)
agent-browser eval 'document.querySelector(".toolkit-trigger-btn").click()'
agent-browser wait 800
agent-browser eval 'document.querySelectorAll("input[type=file]").length'   # confirm: should be 1
agent-browser upload "input[type=file]" /absolute/path/to/file.md           # use CSS selector, NOT @eN ref
agent-browser wait 4000
agent-browser snapshot -i                                # confirm file chip now shows filename + size
```

After the upload **immediately close the lingering popovers** (see Step 4 — there's a persistent `prompt-modal` that will block subsequent clicks if you don't).

### Step 4 — Close lingering popovers, then pick category / theme / page count

**Critical first move**: clicking the `+` button in Step 3 leaves a persistent `prompt-modal` (常用语) panel open that **overlays the lower controls** and silently blocks clicks on `自动页数 / 商业洞察 / 主题卡` — Escape doesn't close it. Hide it via JS before doing anything else:

```bash
agent-browser eval 'document.querySelectorAll(".prompt-modal, .n-popover").forEach(el => el.style.display = "none")'
```

#### 4a. Category (类别)

Default is `通用`. For consulting-style requests, switch to `商业洞察` — this changes which themes are offered:

```bash
agent-browser eval '(() => {
  const t = Array.from(document.querySelectorAll(".line-tab")).find(el => el.textContent.trim() === "商业洞察");
  t?.click(); return t ? "ok" : "not found";
})()'
```

#### 4b. Page count (页数)

Default is `自动页数`. To set explicit count, click the `.page-limit-button`, then pick the bucket:

```bash
agent-browser eval 'document.querySelector(".page-limit-button").click()'
agent-browser wait 600
agent-browser eval '(() => {
  const item = Array.from(document.querySelectorAll("*")).find(el => el.textContent?.trim() === "16-20 页" && el.children.length === 0);
  item?.click(); return item ? "ok" : "not found";
})()'
agent-browser eval 'document.querySelector(".page-limit-button span")?.textContent'   # verify
```

Buckets are coarse (`1-5 / 6-10 / 11-15 / 16-20 / 21-25 / 26-30`) — pick the bucket containing the user's target.

#### 4c. Theme (风格)

⚠️ **AX tree is misleading**: every theme card shows `已选择 蔚蓝冲击 / 已选择 铅灰未来 / ...` — the `已选择` is just a label prefix, not selection state. **Source of truth is the `.style-card.selected` class.** Verify before and after:

```bash
agent-browser eval 'Array.from(document.querySelectorAll(".style-card.selected")).map(el => el.textContent.trim())'
```

To switch theme (e.g. McKinsey → `藏蓝铜金`):

```bash
agent-browser eval '(() => {
  const c = Array.from(document.querySelectorAll(".style-card")).find(el => el.textContent.includes("藏蓝铜金"));
  c?.click(); return c ? "ok" : "not found";
})()'
```

### Step 5 — Type the prompt and kick off generation

#### 5a. Type the prompt

Get the textbox ref from a fresh snapshot (it's the `textbox [ref=eN]` inside `.chat-editor`), then type:

```bash
agent-browser type @e<textbox> "your full prompt here..."
agent-browser eval '(() => document.querySelector(".chat-editor [contenteditable=true]").innerText.length)()'   # confirm length
```

#### 5b. Vue trick: wake up the disabled send button

`agent-browser type` inserts text in a paste-style way that **does not fire Vue's input event**, so `.send-button-container` stays `disabled` even with text in the box. Fix: append one more character via real keypress:

```bash
agent-browser eval 'String(document.querySelector("[class*=send-button]").className)'
# If output still contains "disabled":
agent-browser type @e<textbox> "."
agent-browser eval 'String(document.querySelector("[class*=send-button]").className)'
# Should now be "send-button-container" (no "disabled")
```

#### 5c. Click send

There is **no text-labeled "生成" button** — Kimi uses an icon-only send button. Click it via class:

```bash
agent-browser eval 'document.querySelector("[class*=send-button]").click()'
agent-browser wait 3000
agent-browser snapshot -i      # confirm new chat session named after the topic appears
```

### Step 6 — Wait patiently — generation takes minutes

Kimi renders slides one by one. Do not spam snapshots — that burns tokens and the page stutters. Strategy:

- First **60s**: `agent-browser wait 60000`, then snapshot once.
- If still rendering: another `wait 120000` (2 min), snapshot again.
- Repeat until you see a completion signal: a download/导出/export button, a thumbnail carousel of finished slides, or a "生成完成" banner.
- Hard ceiling: **15 minutes total**. If nothing completes by then, snapshot any error banner and surface it.

Between waits, a short progress ping to the user every ~3 minutes is friendly: "Kimi 还在渲染，已经 4 分钟了，继续等。"

Do **not** use a `sleep`-based polling loop at the bash level — use `agent-browser wait <ms>` and keep the number of Bash turns low.

### Step 7 — Download the PPTX (via the editor iframe)

When generation finishes, the chat shows a result card titled `<topic>` with subtitle `点击编辑和下载` and an action button `去编辑`. **The download flow is NOT in the chat page** — it's inside an iframe `.ppt-frame` (same-origin) that loads the actual editor.

#### 7a. Open the editor iframe

The `去编辑` button reveals the iframe but the chat URL doesn't change. Easiest path: just operate the iframe contentDocument directly (it's already loaded).

```bash
agent-browser eval '(() => {
  const f = document.querySelector(".ppt-frame");
  return { hasIframe: !!f, src: f?.src };
})()'
```

If `hasIframe` is false, click the `去编辑` button:

```bash
agent-browser eval '(() => {
  const btn = Array.from(document.querySelectorAll("*")).find(el => el.textContent?.trim() === "去编辑" && el.children.length < 3);
  btn?.click(); return btn ? "ok" : "not found";
})()'
agent-browser wait 3000
```

#### 7b. Click 导出 inside the iframe

```bash
agent-browser eval '(() => {
  const doc = document.querySelector(".ppt-frame").contentDocument;
  const btn = Array.from(doc.querySelectorAll("[class*=button]")).find(b => b.textContent?.trim() === "导出");
  btn?.click(); return btn ? "ok" : "not found";
})()'
agent-browser wait 1500
```

This opens an `export-dialog` modal inside the iframe. Default format is **`PPT`** (yes, "PPT" — but the file actually saves as `.pptx`). Alternatives: `图片`. No "PPTX" label exists.

#### 7c. Click 直接下载

```bash
agent-browser eval '(() => {
  const doc = document.querySelector(".ppt-frame").contentDocument;
  const btn = Array.from(doc.querySelectorAll("[class*=button]")).find(b => b.textContent?.trim() === "直接下载");
  btn?.click(); return btn ? "ok" : "not found";
})()'
agent-browser wait 8000
```

(Sibling button is `选择目录` — opens a directory picker. We use `直接下载` which saves to `~/Downloads/`.)

#### 7d. Resolve the file

```bash
ls -t ~/Downloads/*.pptx 2>/dev/null | head -1
```

Kimi names it after the deck title (e.g. `中国制造业研究报告（2024–2026 Q1）.pptx`). If nothing appears within 30s, re-check the iframe for a paywall or error banner.

### Step 8 — Move & deliver

Copy into the workspace so it is easy to find later:

```bash
mkdir -p ~/.cctb/default/workspace/kimi-ppt
cp "<newest_downloads_path>" ~/.cctb/default/workspace/kimi-ppt/<slug>-$(date +%Y%m%d-%H%M).pptx
```

Then deliver via the Telegram file tag in your reply:

```
[send-file:/Users/cloveric/.cctb/default/workspace/kimi-ppt/<slug>-YYYYMMDD-HHMM.pptx]
```

Follow with a one-line summary: filename + slide count if you could read it from the final snapshot.

## Error Handling — precise diagnosis, no hand-waving

Always name which branch you hit:

1. **Connect failed** — `agent-browser connect 9222` errored. Surface the error, stop.
2. **Wrong session** — connected, but title/url look like a fresh/empty Chrome. Stop and flag — do not press on.
3. **Not logged in** — login panel visible on kimi.com. Stop, ask user to log in manually.
4. **Verification wall** — captcha or anti-abuse page. Stop, surface what you saw.
5. **Upload rejected** — surface Kimi's error verbatim (file too big, wrong type, etc.).
6. **Generation failed / timed out** — snapshot the error banner, surface it, offer to retry.
7. **Download gated** — if PPTX export requires a paid tier, surface the gate text, do not attempt to bypass.

## What NOT to do

- Do not launch a new Chrome / use chrome-devtools MCP / use Playwright MCP.
- Do not generate the PPT locally — this skill is specifically a wrapper for Kimi's service.
- Do not automate login, captcha, or verification flows.
- Do not click publish / 分享 / post-to-social buttons.
- Do not say "登不了" without telling the user which of the 4 failure branches you hit.
- Do not run a tight polling loop of snapshots — use `agent-browser wait <ms>`.

## Known Kimi UI Quirks — Cheat Sheet

When the AX tree / `find text` / `click @eN` aren't working, it's usually one of these:

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | Click on `自动页数 / 商业洞察 / theme card` does nothing | Persistent `.prompt-modal` (常用语) panel overlays controls; Escape can't dismiss it | `agent-browser eval 'document.querySelectorAll(".prompt-modal, .n-popover").forEach(el => el.style.display = "none")'` |
| 2 | `agent-browser upload @e<LabelText>` → `Node is not a file input element` | The `LabelText "文件和图片"` ref is not the input itself; the actual `<input type=file>` is its hidden child and only exists after you click the `+` button | First click the `+` (`.toolkit-trigger-btn`), then `agent-browser upload "input[type=file]" /path` (CSS selector, not `@eN`) |
| 3 | Send button stays `.send-button-container.disabled` even though prompt textbox shows full text | `agent-browser type` inserts paste-style; Vue's `input` event never fires | Append one more char with `agent-browser type @e<textbox> "."` — fires real keypress, Vue syncs, button enables |
| 4 | All theme cards show `已选择 X` in AX names — looks like all selected | `已选择` is a hard-coded label prefix, not state | Read `Array.from(document.querySelectorAll(".style-card.selected")).map(el => el.textContent)` for the truth |
| 5 | No "生成" / "开始生成" text button anywhere | Send is an icon-only div with class `.send-button-container` | Click via `document.querySelector("[class*=send-button]").click()` |
| 6 | `find text "PPTX"` → nothing | The format radio is labeled `PPT` (the file is still `.pptx`); option labels are `PPT` and `图片` | Default is `PPT` already — just click `直接下载` |
| 7 | Result card click does nothing visible; URL doesn't change | The PPT editor is in a same-origin `iframe.ppt-frame`, not a new page | Operate via `document.querySelector(".ppt-frame").contentDocument.querySelector(...)` |
| 8 | Page count picker — `find text "16-20 页" click` doesn't fire | Dropdown lives in a portal that may close on the next snapshot | `eval` the open-then-pick chain in one shot: open `.page-limit-button`, then click the matching item by textContent |
| 9 | Clicked the "+" button → nothing happened, file input never appeared | Picked the wrong nameless control: snapshot shows two unnamed controls — sidebar `button "添加联系人"` (not it) AND chat-editor's `generic clickable` (the real `+`). Don't grep for "button" | Always click via stable class: `agent-browser eval 'document.querySelector(".toolkit-trigger-btn").click()'` |

## agent-browser Cheatsheet (in order of use here)

```bash
agent-browser connect 9222          # REQUIRED FIRST STEP
agent-browser get title
agent-browser get url
agent-browser open <url>
agent-browser wait --load networkidle
agent-browser snapshot -i           # accessibility tree w/ @eN refs
agent-browser click @eN
agent-browser find text "..." click # click by visible text (less reliable on Kimi than eval)
agent-browser type @eN "text"       # for prompt input — fires real keypresses
agent-browser upload <selector> /abs/path  # use CSS selector when @eN is a wrapper, not the input
agent-browser eval '<JS>'           # heavy lifter for Kimi: hide modals, query .selected state, click via class, drive the iframe
agent-browser wait <ms>             # long wait between polls
agent-browser screenshot out.png    # sanity check only; daemon may be busy if used right after eval/click
```

Refs (`@e1`, `@e2`…) are reassigned on every snapshot. Re-snapshot after any page change before using refs again. **For Kimi specifically, `eval` with class-based queries is more reliable than `@eN` refs because Kimi's controls are Vue components in portals/popovers that the AX tree often misrepresents.**
