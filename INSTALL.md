# Installation

## 1. Install Claude Code

Download from [claude.ai/code](https://claude.ai/code) and sign in with your Anthropic account.

## 2. Copy the skill

Copy `SKILL.md` into your Claude skills directory:

```bash
mkdir -p ~/.claude/skills/ui-flow-test-case
cp SKILL.md ~/.claude/skills/ui-flow-test-case/SKILL.md
```

**Where is `~/.claude/`?**

| OS | Path |
|----|------|
| macOS | `/Users/<your-username>/.claude/` |
| Windows | `C:\Users\<your-username>\.claude\` |
| Linux | `/home/<your-username>/.claude/` |

If the `skills/` folder does not exist yet, the `mkdir -p` command above creates it.

## 3. Install Python dependencies

Open a terminal and run:

```bash
pip3 install reportlab Pillow openpyxl
```

These are needed to generate the User Flow Document PDF and the Test Specification Excel.
Claude will remind you if any are missing when you run the skill.

## 4. Add the Figma MCP server

The Figma MCP server must be registered in your Claude Code config before the skill
can read any Figma file. It connects via a hosted URL — **no Node.js, no npm, no local
process is needed.**

> **If you already see a `figma` entry in your config that uses `"command": "npx"` —
> that is the old local-server format. Replace it with the HTTP entry below, otherwise
> you will get an `ENOENT: npx not found` error even if everything else is correct.**

**Option A — one command using Python (macOS / Linux / Windows):**

```bash
python3 -c "
import json, os, pathlib
f = pathlib.Path.home() / '.claude.json'
cfg = json.loads(f.read_text()) if f.exists() else {}
key = os.getcwd()
cfg.setdefault(key, {}).setdefault('mcpServers', {})['figma'] = {
    'type': 'http',
    'url': 'https://mcp.figma.com/mcp'
}
f.write_text(json.dumps(cfg, indent=2))
print('Figma MCP server added (HTTP mode).')
"
```

**Option B — manual:**

Open `~/.claude.json` in any text editor. Find the entry for your project directory
(or the global `{}` key) and set the `figma` entry under `mcpServers` to exactly this —
replacing any existing `figma` entry if one is already there:

```json
"mcpServers": {
  "figma": {
    "type": "http",
    "url": "https://mcp.figma.com/mcp"
  }
}
```

**After adding or updating the entry, restart Claude Code.**

Open Claude Code → Settings → MCP Servers and confirm `figma` shows status **Connected**.
If it shows **Disconnected**, make sure you are logged in to Figma in your browser —
the MCP server authenticates via your Figma browser session automatically.

No personal access token is needed.

## 5. Verify the skill is installed

Open Claude Code and type:

```
/ui-flow-test-case
```

You should see the skill activate and ask for a Figma URL.

Alternatively, just paste a Figma prototype URL in the chat:

```
Generate a flow doc and test spec for this prototype:
https://www.figma.com/proto/...
```

---

## Output location

By default, all generated files are saved to:

```
~/Desktop/claude/
```

This folder is created automatically if it does not exist.

---

## Troubleshooting

**"Please open this URL in your browser to authorize the Figma connection"**
This is the normal first-time login — not an error. Open the link, log in with your Figma
(or SAP SSO) account, then copy the full redirect URL from the browser address bar and paste
it back into Claude. The page it lands on may look broken — the URL is what matters.
This only happens once per session.

**"ENOENT: Executable not found in \$PATH: npx" or "npx not found"**
Your `~/.claude.json` has a `figma` entry that uses the old `npx`-based local server
format. This requires Node.js and fails if Node.js is not installed. The HTTP server
used by this skill does not need Node.js at all. Fix it by replacing your existing
`figma` entry with the HTTP version from step 4 above, then restart Claude Code.
You do **not** need to install Node.js.

**"No module named 'reportlab'"**
Run: `pip3 install reportlab Pillow openpyxl`

**"Figma MCP not connected"**
Open `~/.claude.json` and confirm the `figma` entry uses `"type": "http"` (see step 4).
Restart Claude Code after any change to `~/.claude.json`. Also check that you are
logged in to Figma in your browser.

**"Skill not found when I type /ui-flow-test-case"**
Check that `SKILL.md` is at exactly this path:
`~/.claude/skills/ui-flow-test-case/SKILL.md`
The folder name must match the `name:` field in the SKILL.md frontmatter.
