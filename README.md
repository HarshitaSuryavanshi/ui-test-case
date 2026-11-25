# UI Flow → Test Case

A Claude Code skill for **developers** who automate UI testing from Figma prototypes.

Point it at a Figma prototype URL. It captures every screen in the flow — always fresh, never cached — and produces:

1. **User Flow Document PDF** — one page per screen with the actual Figma screenshot, plus STATE / USER ACTION / SYSTEM RESPONSE / LEADS TO. Feed this into Claude + Playwright to generate test scripts.
2. **Test Specification Excel** — Given/When/Then test cases with Playwright selector hints, priority, and a Definition of Done checklist.

No text review. No compliance checking. Just screens → tests.

---

## Who this is for

Frontend developers and QA engineers who:
- Receive Figma prototypes and need to write Playwright (or Cypress) UI tests
- Want to use Claude to auto-generate test scripts from a flow document
- Need a reproducible, always-up-to-date capture of the prototype — not a stale screenshot from last sprint

---

## Prerequisites

**1. Claude Code**
Download from [claude.ai/code](https://claude.ai/code) and sign in.

**2. Figma MCP server**
Included with Claude Code. Make sure you are logged in to Figma in your browser — the MCP server uses your Figma account session automatically. No personal access token needed.

**3. Python 3 with dependencies**
```bash
pip3 install reportlab Pillow openpyxl
```

---

## Installation

Copy `SKILL.md` into your Claude skills directory:

```bash
mkdir -p ~/.claude/skills/ui-flow-test-case
cp SKILL.md ~/.claude/skills/ui-flow-test-case/SKILL.md
```

That's it. No other skill dependencies — this skill is fully self-contained.

---

## How to use

1. Open Claude Code in your project directory
2. Type `/ui-flow-test-case` or just paste a Figma URL:
   > "Generate a flow doc and test spec for this prototype: https://www.figma.com/proto/..."
3. Claude captures all screens fresh from Figma and generates both output files automatically

---

## Output files

Both files are saved to `~/Desktop/claude/` by default.

| File | What it's for |
|------|--------------|
| `<FlowName>_UserFlowDocument.pdf` | Attach to Claude and prompt it to write Playwright specs |
| `<FlowName>_TestSpecification.xlsx` | Given/When/Then test cases with Playwright selector hints |

---

## Generating Playwright tests from the output

Once the User Flow Document PDF is generated:

1. Open Claude Code in your Playwright project
2. Attach the PDF and run:
   > "Generate Playwright test specs from this flow document. Use the Given/When/Then
   > structure. Output one `describe()` block per screen, with `beforeEach` navigation
   > and `expect()` assertions for each Then statement."

The Test Specification Excel also contains a **Playwright Setup** sheet with a starter
`playwright.config.ts` snippet and a `beforeEach` hook.

---

## Why always fresh?

The skill never reuses cached screen data. Every run re-fetches every frame directly from
Figma. This means:

- New screens added since the last run are captured automatically
- Removed screens are no longer included
- Changed labels, states, or flows are reflected immediately

If your prototype changes between sprints, just re-run the skill — you get the current
state without having to manage what's changed.

---

## Figma personal access token

Not needed. The Figma MCP server handles authentication via your Figma browser session.

---

## Author

Harshita Suryavanshi — User Assistance, SAP Integration Suite  
harshita.suryavanshi@sap.com
