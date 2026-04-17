# gimme-ppt-skill

A [Claude Code](https://claude.ai/code) skill that generates a PowerPoint deck by driving [kimi.com/slides](https://www.kimi.com/slides) in your **already-logged-in Chrome**, then downloads and delivers the `.pptx`.

It is a **wrapper around a real browser session**, not an API reimplementation — Kimi's auth cookies, login state, and any paid-tier entitlements come from the Chrome you're already using.

## Why

- Kimi's slide generator produces genuinely nice decks (layout variety, chart suggestions, image search, 10–20+ pages) without you having to write a single line of PptxGenJS.
- A normal Claude Code "make a PPT" skill runs locally and has to fake all of that.
- But kimi.com/slides needs a logged-in session, so a fresh headless browser can't use it.
- This skill threads the needle: Claude drives the browser *you* already have open and logged in.

## How it works

```
  ┌──────────────────────────┐
  │ Your running Chrome      │   launched with --remote-debugging-port=9222
  │ (logged into kimi.com)   │
  └──────────┬───────────────┘
             │ CDP
  ┌──────────▼───────────────┐
  │ agent-browser CLI        │   the only approved driver
  └──────────┬───────────────┘
             │
  ┌──────────▼───────────────┐
  │ Claude Code + this skill │   open page, upload doc, pick layout/style,
  │                          │   wait patiently, click 编辑和下载 →
  │                          │   iframe 导出 → PPT → 直接下载,
  │                          │   then deliver the .pptx
  └──────────────────────────┘
```

## Requirements

1. **Chrome** running with remote debugging on port 9222:
   ```bash
   # macOS example — adjust --user-data-dir to a profile that is logged into kimi.com
   /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
     --remote-debugging-port=9222 \
     --user-data-dir="$HOME/.chrome-kimi-profile"
   ```
   Log into [kimi.com](https://www.kimi.com) in that window **once** before first use.

2. **[agent-browser](https://www.npmjs.com/package/agent-browser)** CLI on your `$PATH`:
   ```bash
   npm i -g agent-browser
   # or: bun add -g agent-browser
   agent-browser --help
   ```

3. **[Claude Code](https://claude.ai/code)** — this is a Claude Code skill, so you need the CLI (or any compatible harness that loads `~/.claude/skills/`).

## Install

Copy this folder into your Claude Code skills directory:

```bash
git clone https://github.com/cloveric/gimme-ppt-skill.git
mkdir -p ~/.claude/skills
cp -r gimme-ppt-skill ~/.claude/skills/kimippt
```

(The skill is named `kimippt` internally so its trigger-aliases match that name. You can fold it under a different directory name if you prefer, but Claude Code keys off the `name:` field inside `SKILL.md`.)

## Usage

In your Claude Code session, attach a `.docx` / `.pdf` / `.md` and type one of the trigger phrases:

- `kimippt`
- `/kimippt`
- `帮我生成kimippt` / `帮我用kimi生成ppt`
- `kimi ppt` / `kimi slides`

Optional overrides in the same message:

- Style — e.g. "用商务风", "换成极简", "金色系"
- Page count — e.g. "20 页"
- Tone — e.g. "咨询风格，信息量充分"

Defaults are `智能布局` (smart layout) + `自由风格` (free style).

The skill will:

1. Connect to Chrome on `127.0.0.1:9222`, verify it's your real session (not a fresh profile).
2. Open `kimi.com/slides`, diagnose login / captcha / logged-in state.
3. Upload the document.
4. Pick layout + style (default or your override).
5. Click generate, then wait patiently — Kimi can take **5–15 minutes** for a 20-page deck.
6. When the editor iframe shows `📊 演示文稿完成情况`, click 编辑和下载 → 导出 → PPT → 直接下载.
7. Copy the newest `.pptx` from `~/Downloads/` into a workspace folder and surface the absolute path.

## Things this skill will NOT do

- Launch a fresh browser (would be logged out).
- Use `chrome-devtools` MCP or Playwright MCP (would be a separate session — same logged-out problem).
- Automate login, captcha, or anti-abuse flows.
- Click publish / 分享 / post-to-social buttons.
- Pretend success without actually downloading — if the download affordance never appears within the 15-minute ceiling, the skill surfaces the error banner.

## Known fragilities

- **cwd can matter for `agent-browser`** — if the daemon returns empty output from your project directory, `cd /tmp &&` before the call. This is an upstream quirk, not something this skill can fix.
- **Chrome must actually be the profile that is logged into Kimi.** If you have multiple Chrome profiles, make sure the one with `--remote-debugging-port=9222` is the logged-in one.
- **iframe editor** — the 导出 / 下载 buttons live inside a same-origin iframe pointing at `kimi.com/ppt/`. The skill uses `iframe.contentDocument` via `agent-browser eval` to click them. If Kimi changes to cross-origin or adds a shadow DOM, this will need updating.
- **Long waits** — the skill deliberately uses `agent-browser wait <ms>` instead of tight polling, to avoid hammering the page. Be patient.

## Troubleshooting

The skill's error handling names which of these branches it hit — use the message to localize the failure:

| Branch | Meaning | Action |
|---|---|---|
| Connect failed | `agent-browser connect 9222` errored | Check Chrome is running with `--remote-debugging-port=9222` and no other process holds 9222 |
| Wrong session | Connected but on a fresh/empty Chrome | You pointed at the wrong profile — relaunch the logged-in one |
| Not logged in | Login panel visible on kimi.com | Log in manually once in that Chrome window, then retry |
| Verification wall | Captcha or "异常访问" | Solve the captcha in the Chrome window, then retry |
| Upload rejected | Kimi's error (file too big, wrong type) | Use a supported format (.docx / .pdf / .md), under Kimi's size cap |
| Generation timed out | Nothing completed in 15 min | Retry — Kimi occasionally stalls. Consider a smaller page count |
| Download gated | PPTX export is paywalled for your account | Upgrade in Kimi or switch to an account with export entitlement |

## License

MIT. Use at your own risk — this skill is a community automation of a third-party product, not an official Kimi integration. Respect [Kimi's Terms of Service](https://www.kimi.com/).

## Not affiliated with Moonshot AI / Kimi

This project is a community skill. It does **not** use any private Kimi API, does **not** bundle Kimi credentials, and does **not** attempt to bypass paywalls or anti-abuse measures. It simply drives a browser you already control.
