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

3. **Resolve the input document path.** Use the absolute path from the user's message verbatim (e.g. a file they uploaded through your harness). Do not re-fetch or re-upload.

## Default Settings

Unless overridden in the same message:

- **Layout (布局)** — `智能布局`
- **Style (风格)** — `自由风格`

If the user writes e.g. "用商务风", "换成极简" — pick that style instead. Everything else (page count, theme color) stays default.

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

Kimi's page exposes either a visible "上传" button or a drop zone with a hidden `<input type="file">`. Workflow:

```bash
agent-browser snapshot -i            # find the upload control
agent-browser upload @eN /absolute/path/to/file.docx
agent-browser wait 3000              # let the client ack the file
agent-browser snapshot -i            # confirm filename now visible
```

If clicking the visible button first is required to reveal the file input, do that with `click @eN`, then re-snapshot and `upload`.

### Step 4 — Pick layout + style

After the file is accepted, layout + style pickers should be visible. Re-snapshot and:

```bash
agent-browser find text "智能布局" click
agent-browser snapshot -i
agent-browser find text "自由风格" click       # or the user-specified override
agent-browser snapshot -i
```

If these options only appear after a "下一步" step, handle that transition first, then re-snapshot and click.

### Step 5 — Kick off generation

Click the final generate/创建 button (common labels: `生成`, `开始生成`, `创建 PPT`). Record the start time.

```bash
agent-browser find text "生成" click     # pick the button, not a heading
agent-browser snapshot -i
```

### Step 6 — Wait patiently — generation takes minutes

Kimi renders slides one by one. Do not spam snapshots — that burns tokens and the page stutters. Strategy:

- First **60s**: `agent-browser wait 60000`, then snapshot once.
- If still rendering: another `wait 120000` (2 min), snapshot again.
- Repeat until you see a completion signal: a download/导出/export button, a thumbnail carousel of finished slides, or a "生成完成" banner.
- Hard ceiling: **15 minutes total**. If nothing completes by then, snapshot any error banner and surface it.

Between waits, a short progress ping to the user every ~3 minutes is friendly: "Kimi 还在渲染，已经 4 分钟了，继续等。"

Do **not** use a `sleep`-based polling loop at the bash level — use `agent-browser wait <ms>` and keep the number of Bash turns low.

### Step 7 — Download the PPTX

When the download affordance appears:

```bash
agent-browser snapshot -i
# Locate the download/export button and the format picker
agent-browser click @eN           # opens download menu
agent-browser snapshot -i
agent-browser find text "PPTX" click
```

Kimi saves to Chrome's default download directory (typically `~/Downloads/`). Resolve the newest `.pptx`:

```bash
ls -t ~/Downloads/*.pptx 2>/dev/null | head -1
```

If no file appears within 30s, re-snapshot — Kimi may be finalizing.

### Step 8 — Move & deliver

Copy into a workspace folder so it is easy to find later (adjust `$WORKSPACE` to wherever your harness keeps generated artifacts):

```bash
mkdir -p "$WORKSPACE/kimi-ppt"
cp "<newest_downloads_path>" "$WORKSPACE/kimi-ppt/<slug>-$(date +%Y%m%d-%H%M).pptx"
```

Then deliver the file through whatever mechanism your harness uses — e.g. a Telegram bridge `[send-file:<absolute-path>]` tag, an MCP attachment tool, or just surfacing the absolute path. Follow with a one-line summary: filename + slide count if you could read it from the final snapshot.

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

## agent-browser Cheatsheet (in order of use here)

```bash
agent-browser connect 9222          # REQUIRED FIRST STEP
agent-browser get title
agent-browser get url
agent-browser open <url>
agent-browser wait --load networkidle
agent-browser snapshot -i           # accessibility tree w/ @eN refs
agent-browser click @eN
agent-browser find text "..." click # click by visible text
agent-browser upload @eN /abs/path
agent-browser wait <ms>             # long wait between polls
agent-browser screenshot out.png    # sanity check only, use sparingly
```

Refs (`@e1`, `@e2`…) are reassigned on every snapshot. Re-snapshot after any page change before using refs again.
