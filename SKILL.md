---
name: ui-flow-test-case
author: Harshita Suryavanshi
description: >
  Captures every screen in a Figma prototype flow (always fresh — never cached),
  generates a User Flow Document PDF with STATE / USER ACTION / SYSTEM RESPONSE /
  LEADS TO per screen, and a Playwright-ready Test Specification Excel. Designed for
  developers who automate UI testing from Figma prototypes. Activate with a Figma
  prototype or design URL.
---

# UI Flow → Test Case

This skill is for **developers** who need to generate automated UI tests from a Figma
prototype. It does one thing well: capture every screen in the flow, describe it
precisely, and produce two machine-ready outputs:

1. **User Flow Document PDF** — one page per screen with Figma screenshot, STATE,
   USER ACTION, SYSTEM RESPONSE, and LEADS TO. Use this as the input to Claude +
   Playwright to generate test automation scripts.

2. **Test Specification Excel** — Given/When/Then test cases per screen, one row per
   user action, with Playwright selector hints and acceptance criteria. Import directly
   into your test suite or use it to prompt Claude to write Playwright specs.

---

**No comments, no text review, no PDF report.** This skill captures screens and
produces developer handoff documents only. All analysis is done in memory — no
intermediate files are created.

---

## Workflow

### Step -1 — Prerequisites Check

**Run this silently before anything else — before asking any questions, before fetching any Figma data.**

#### 1. Figma MCP connection

Attempt any Figma MCP tool call. If it succeeds, the MCP server is connected — proceed
silently.

If the Figma MCP server is not connected or returns any error:

**CRITICAL — do NOT suggest any of the following. These are wrong for this setup:**
- Do NOT suggest `claude plugin install figma`
- Do NOT suggest adding `"command": "npx"` or `"args": ["-y", "@figma/mcp-server"]` to any config
- Do NOT ask for or suggest a Figma Personal Access Token to fix the connection
- Do NOT suggest installing Node.js

**The only correct fix is to register the HTTP server entry.** Show this message exactly:

```
The Figma MCP server is not connected.

Do NOT install any plugin or use npx — those require Node.js and a token, and are not needed here.

The correct fix is a one-line config entry. Run this in your terminal:

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
  print('Done.')
  "

If that does not work, open ~/.claude.json manually and add or replace the figma entry
under mcpServers for your project path:

  "figma": {
    "type": "http",
    "url": "https://mcp.figma.com/mcp"
  }

This uses a hosted server — no Node.js, no token, no plugin needed.
Restart Claude Code after saving, then run the skill again.
```

Do not proceed past this point until the Figma MCP connection is confirmed working.



#### 2. Python dependencies

Run:
```bash
python3 -c "import reportlab, PIL, openpyxl" 2>/dev/null && echo "OK" || echo "MISSING"
```

If the output is `MISSING`, run the install command automatically:
```bash
pip3 install reportlab Pillow openpyxl -q
```

Confirm installation succeeded before continuing. If `pip3` itself is not found, show:
```
Python packages are missing. Please run this in your terminal, then try again:
  pip3 install reportlab Pillow openpyxl
```

---

### Step 0 — Always Fresh Capture

**Every run is a fresh capture. No exceptions.**

Do NOT ask the user whether to reuse session data. Do NOT carry over any screen names,
node IDs, descriptions, or screenshots from any prior session or context — even if the
flow looks identical to one captured before.

**Always:**
- Re-fetch every frame from Figma via `get_metadata` and `get_screenshot`
- Re-traverse the prototype graph from the entry node
- Re-download every screenshot

This ensures the developer always gets the current state of the prototype, including
any new screens, removed screens, or changed content since the last run.

If the prototype has not changed and the developer just wants to re-generate outputs,
re-capture anyway — it takes seconds and guarantees accuracy.

---

### Step 1 — Resolve the Figma Target

Accept any of:
- A **prototype URL**: `https://www.figma.com/proto/<fileKey>/...?node-id=<startNodeId>`
- A **design page URL**: `https://www.figma.com/design/<fileKey>/...?node-id=<pageId>`
- A **Figma Make URL**: `https://www.figma.com/make/<makeFileKey>/...`

Extract `fileKey` and `nodeId` from the URL.

**Figma Make URLs (`/make/`):**
- Extract `makeFileKey` from the URL path — use it as `fileKey` for all MCP tool calls.
- Assume `nodeId = 0:1` (the root node) when no `node-id` parameter is present in the URL.
- Use `get_design_context` with `nodeId: "0:1"` to read the Make file — this is the correct entry point for Make files.
- Do NOT use `get_metadata` alone on Make files — always use `get_design_context`.

**OAuth login prompt (expected behaviour):**
If Claude shows a browser authorisation link (`https://www.figma.com/oauth/mcp?...`) before fetching any Figma data, this is the normal first-time login flow — not an error. The user must:
1. Open the link in their browser and log in with their Figma / SAP SSO credentials.
2. After the redirect, copy the full URL from the browser address bar (even if the page looks broken) and paste it back into Claude.
The connection is then active for the session. This prompt only appears once per session or after the token expires.

**If the user specifies a section name** (e.g. "only the ESR Migration section"):
- Call `get_metadata` on the page node to get the full canvas structure
- Find the `<section>` node whose name matches the user's description
- Collect only `<frame>` nodes that are direct or nested children of that section
  with `width="1440"` (standard screen width)
- Ignore frames with `hidden="true"`

**Prototype URL — Graph Traversal (BFS, handles branching):**

```
visited  = {}          # nodeId → { name, depth, path_label, branch_label }
queue    = [(startNodeId, depth=0, path="Main", branch_label="Main Path")]
```

For each node dequeued:
1. If already in `visited`, skip (prevents infinite loops).
2. Call `get_metadata` on the nodeId.
3. Parse all `destinationId` prototype link targets from the metadata.
4. For each destination not yet visited:
   - First outgoing link from this node: same path label
   - Two or more outgoing links: branch point — label each `Branch A`, `Branch B`, etc.
   - Enqueue with incremented depth and resolved path/branch label
5. Add node to `visited`.

After traversal, sort by depth then path label.

**Cycle / depth guard:** stop if depth > 30 or total nodes > 100 — log a warning and
list unreached nodes so the developer can investigate.

**Design page URL (no prototype links):**
- Call `get_metadata` on the page
- Collect all top-level `<frame>` nodes with `width="1440"`, excluding hidden and
  utility frames

Store the ordered list of frame IDs, names, and path labels. This is the **frame inventory**.

---

### Step 2 — Screenshot All Frames

For each frame in the inventory:
- Call `get_screenshot` with `maxDimension: 2048`
- Download immediately: `curl -s -L -o /tmp/uiflow_frame_<n>.png "<url>"`

Batch downloads in parallel where possible.

**What to record per frame** (from screenshot + metadata):
- All visible text: titles, labels, buttons, tabs, column headers, messages,
  placeholders, breadcrumbs, section headings, status values
- UI element types (button, input, dropdown, table, dialog, wizard step indicator,
  empty state, loading state, error message)
- Interactive elements: which buttons/links are clickable, which fields accept input
- Visible state: which tab is active, which step is highlighted, what data is shown
- Screen dimensions and layout zone (full page / dialog / panel / wizard)

Do NOT extract: sample data values in table rows, lorem ipsum, technical IDs used
as data values.

Store per frame:
```
Frame N — <name> [<path label>]
  Node ID: <id>
  Visible texts: [list]
  Interactive elements: [list of {element, label, type}]
  Current state description: <1–2 sentences>
```

---

### Step 3 — Build Flow Descriptions

For each frame, derive the four flow dimensions from the screenshot and metadata:

**STATE** — What is currently shown on screen. Present tense. Describe the visible
state: which wizard step is active, what data is displayed, what mode the UI is in.

**USER ACTION** — The primary action the user takes on this screen. One action only.
Use device-neutral verbs: Choose, Select, Enter, Review, Confirm. If the screen is
purely informational (review/summary), the action is "Reviews [content] and chooses
[button]".

**SYSTEM RESPONSE** — What the system does in reaction to the user action. What
changes: navigation, data loading, validation, state transition.

**LEADS TO** — Frame N — `<next screen name>`, or for branch points list each:
  - Choose [Button A] → Frame N — `<screen A>`
  - Choose [Button B] → Frame N — `<screen B>`

For the last screen: "End of flow."

**Branch labelling:**
- Main path: `Screen N — <Frame Name>`
- Branch: `Screen N — <Frame Name>  [Branch from Screen M — <condition>]`

---

### Step 4 — Generate User Flow Document PDF

**File name:** `<FlowName>_UserFlowDocument.pdf`
**Output path:** `~/Desktop/claude/<FlowName>_UserFlowDocument.pdf`

Install if needed:
```bash
pip3 install reportlab Pillow -q
```

Write a Python script to `/tmp/gen_uiflow_flowdoc.py` and run it.

**Document structure:**

| Page | Contents |
|------|----------|
| 1 | Cover |
| 2 | Flow Overview table |
| 3–N | One page per screen |

---

**Cover page:**
- Top band: `#0a4a8c` → `#0064d9` gradient effect (two rectangles)
- Title: flow name, large white bold
- Subtitle: product / feature name
- Meta block: Date, Prepared by, Screen count, Figma file key
- Purpose box (light grey `#F5F6F7`):
  > "This document describes the complete UI flow captured from Figma. It is intended
  > for use with Claude + Playwright to generate automated UI test scripts. Each screen
  > page contains the actual Figma screenshot, a precise description of the current
  > state, the user action taken, the system response, and the next screen in the flow."
- Footer: `<FlowName> User Flow Document  ·  <Date>  ·  Page 1`

---

**Flow Overview table (page 2):**

Columns: `#` | `Screen Name` | `Path` | `User Action Summary` | `Leads To`

- Blue `#0064d9` column headers, white text
- Alternating row colours: white / `#F0F4F9`
- Branch path labels in orange `#E76500`
- Footer: same pattern as cover

---

**Per-screen page layout (A4 portrait):**

```
┌──────────────────────────────────────────────────────────────┐
│  Screen N of Total   <Screen Name>              <Path Label> │  ← banner
├──────────────────────────────────────────────────────────────┤
│                                                              │
│            [SCREENSHOT — full width, max 42% of page]       │
│                                                              │
├──────────────┬───────────────────────────────────────────────┤
│  STATE       │  <what is currently shown on screen>         │
├──────────────┼───────────────────────────────────────────────┤
│  USER ACTION │  <what the user does>                        │
├──────────────┼───────────────────────────────────────────────┤
│  SYSTEM      │  <how the system responds>                   │
│  RESPONSE    │                                              │
├──────────────┼───────────────────────────────────────────────┤
│  LEADS TO    │  Screen M — <next screen name>               │
│              │  [or branch list]                            │
└──────────────┴───────────────────────────────────────────────┘
  Node ID: <nodeId>
```

**Styling:**
- Main path screens: SAP Blue `#0064D9` banner, white text
- Branch screens: SAP Orange `#E76500` banner, white text
- Label column (STATE / USER ACTION / SYSTEM RESPONSE / LEADS TO): `#0040B0` fill,
  white text, bold
- Body column: white / `#F0F4F9` alternating rows
- Node ID line: `#888888` small font below the table — useful for Playwright selectors
- Footer: `<FlowName> User Flow Document  ·  SAP Integration Suite  ·  Page N`

**Screenshot handling:**
- Use PNGs downloaded in Step 2 at `/tmp/uiflow_frame_<n>.png`
- Scale to fit available width preserving aspect ratio, capped at 42% of page height
- If a screenshot is unavailable: grey placeholder box with the node ID

---

### Step 5 — Generate Test Specification Excel

**File name:** `<FlowName>_TestSpecification.xlsx`
**Output path:** `~/Desktop/claude/<FlowName>_TestSpecification.xlsx`

Install if needed:
```bash
pip3 install openpyxl -q
```

Write a Python script to `/tmp/gen_uiflow_testspec.py` and run it.

**SAP colours for the workbook:**
- Blue: `#0064D9` — headers
- DarkBlue: `#0A4A8C` — cover band, sheet tabs
- Orange: `#E76500` — branch path labels, High priority
- Green: `#0F6E35` — pass status
- Red: `#BB0000` — Critical priority, fail status
- LightGrey: `#F5F6F7` — alternating rows

---

**Sheet 1 — Cover**

| Field | Value |
|-------|-------|
| Title | `<FlowName> — Test Specification` |
| Product | (from user or derived from Figma file name) |
| Feature | (flow name) |
| Date | today |
| Prepared by | Developer Review |
| Figma file | `<fileKey>` |
| Total test cases | (count) |
| Total screens | (count) |

Style: DarkBlue header band, metadata table, summary count boxes.

---

**Sheet 2 — Overview**

Columns: `TC-ID` | `Screen` | `Path` | `Test Type` | `Priority` | `Status`

One row per test case with a one-line description. Priority colour-coded:
- Critical → Red `#BB0000`
- High → Orange `#E76500`
- Medium → `#E9730C` (lighter orange)
- Low → `#0064D9` blue

Status defaults to `Not Started` for all rows.

---

**Sheet 3 — Test Cases** (main sheet)

Columns:
`TC-ID` | `Screen` | `Node ID` | `Path` | `Given` | `When` | `Then` |
`Playwright Hints` | `Priority` | `Acceptance Criteria` | `Notes`

**Deriving test cases from the flow:**

Generate one test case per USER ACTION per screen. Additional test cases for:
- Each branch path (one TC per branch)
- Each validation / error state visible in the prototype
- Each empty state visible
- Each loading / progress state visible

**Given/When/Then rules:**
- **Given**: describe the precondition — what screen the user is on, what state it is in
- **When**: describe the user action — use the same verb as USER ACTION in the flow doc
- **Then**: describe the expected system response — use the SYSTEM RESPONSE from the flow doc

**Playwright Hints column** — populate based on what is visible in the screenshot:
- For buttons: `page.getByRole('button', { name: '<label>' }).click()`
- For inputs: `page.getByLabel('<label>').fill('<value>')`
- For dropdowns: `page.getByRole('combobox', { name: '<label>' }).selectOption('<value>')`
- For navigation assertions: `expect(page).toHaveURL(<pattern>)` or
  `expect(page.getByRole('heading')).toContainText('<title>')`
- For dialogs: `page.getByRole('dialog', { name: '<title>' })`
- If the exact selector cannot be determined from the screenshot alone, write:
  `// Inspect element in browser — suggested: page.getByRole('<role>', { name: '<visible label>' })`

**Priority assignment:**
- Critical — test covers a destructive action (delete, overwrite, cancel with data loss)
- High — test covers a primary user action on the happy path
- Medium — test covers a branch path or optional step
- Low — test covers an informational screen with no user input

---

**Sheet 4 — Screen Map**

Maps screen names to node IDs and test case IDs. Useful for navigating between the
Figma prototype and the test spec.

Columns: `Screen #` | `Screen Name` | `Node ID` | `Figma Link` | `Test Cases`

Figma link format: `https://www.figma.com/design/<fileKey>?node-id=<nodeId>`

---

**Sheet 5 — Playwright Setup**

A reference sheet for the developer — not test cases, but setup code they will need.

Include:
1. Recommended `playwright.config.ts` snippet with baseURL, screenshot on failure,
   trace on first retry
2. A starter `beforeEach` hook that navigates to the entry screen
3. A note on how to use the User Flow Document PDF with Claude:
   > "Open Claude Code, attach the User Flow Document PDF, and prompt:
   > 'Generate Playwright test specs for each screen described in this flow document.
   > Use the Given/When/Then structure and Playwright selector hints from the Test
   > Specification Excel. Output one describe() block per screen.'"

---

**Sheet 6 — Definition of Done**

A checklist the developer signs off before closing the testing story.

Include:
- [ ] All TC-IDs in Sheet 2 have status `Pass` or `Skipped (with reason)`
- [ ] No Critical or High priority test cases have status `Fail`
- [ ] Screenshots captured on failure are attached to the Jira story
- [ ] Playwright trace file is archived for failed tests
- [ ] All branch paths have at least one passing test case
- [ ] Flow document matches the implemented UI (no screen missing, no extra screens)

---

### Step 6 — Confirm and Report

After both files are generated, confirm in chat:

```
Flow capture complete.

Screens captured: N
Branch paths: M

Output files:
  ~/Desktop/claude/<FlowName>_UserFlowDocument.pdf    — User Flow Document (N screens)
  ~/Desktop/claude/<FlowName>_TestSpecification.xlsx  — Test Specification (X test cases)

Next step — generate Playwright tests:
  1. Open Claude Code in your project directory
  2. Type: /ui-flow-test-case or use the prompt below
  3. Attach the User Flow Document PDF and prompt Claude:
     "Generate Playwright test specs from this flow document. Use the Given/When/Then
      structure. Output one describe() block per screen, with beforeEach navigation
      and expect() assertions for each Then statement."
```

---

## Output files

| File | Description |
|------|-------------|
| `<FlowName>_UserFlowDocument.pdf` | Per-screen flow spec with Figma screenshots — the primary input for Playwright generation |
| `<FlowName>_TestSpecification.xlsx` | Given/When/Then test cases with Playwright selector hints |

Both files are saved to `~/Desktop/claude/` by default.

---

## Technical reference

**Figma MCP tools used:**
- `get_metadata` — reads node structure and prototype link targets
- `get_screenshot` — takes a PNG screenshot of each frame at 2048px
- `get_design_context` — reads Make file source file resource URIs (nodeId: "0:1")
- `ReadMcpResourceTool` — reads Make file source files via `file://figma/make/source/<fileKey>/...` URIs

**Python libraries required:**
```bash
pip3 install reportlab Pillow openpyxl
```

**Node ID format:** Figma node IDs use the format `1234:5678`. They appear as `1234-5678`
in URLs — convert the hyphen to a colon when calling MCP tools.

**Playwright version assumed:** v1.40+ (uses `getByRole`, `getByLabel`, `getByText`
locators — not deprecated `$` selectors).

---

## Figma Make file support

Figma Make files (`/make/` URL path) are React/TypeScript apps — they have no prototype
links and no visual frames. The skill works differently on them:

### How to read a Make file

1. **Do NOT call `create_session`** — it is not supported for Make files and will fail with
   "This tool is not supported for Make files." Skip it entirely.

2. **Call `get_design_context` with `nodeId: "0:1"`** — this is the correct entry point.
   It returns a list of source file resource URIs in the form:
   `file://figma/make/source/<makeFileKey>/src/App.tsx`

3. **Read source files via `ReadMcpResourceTool`** — pass the `file://figma/make/source/...`
   URIs returned by `get_design_context`. The primary file is `src/App.tsx`.

4. **Do NOT call `get_screenshot`** — screenshots are not available for Make files.
   Use grey placeholder boxes in the User Flow Document PDF instead.

### How to derive the screen inventory

From `src/App.tsx`:
- **Screen names** — find the TypeScript `type Screen = "landing" | "overview" | ...` union.
  Each string literal is a screen identifier.
- **Navigation graph** — find `useState<Screen>` and all `setCurrentScreen(...)` / `onClick`
  handlers. Each `setCurrentScreen("target")` call is a navigation edge.
- **Screen descriptions** — find the `renderCurrentScreen()` function or similar switch/if
  block. Each case describes what that screen renders.

### PDF generation for Make files

- Use grey placeholder boxes (ReportLab `Rect` with fill `#D8D8D8`) instead of screenshots.
- Include a note on the cover page: "Screenshots are not available for Figma Make files.
  Flow descriptions are derived from the application source code."
- All other PDF structure (cover, flow overview table, per-screen pages) is identical to
  the standard flow for prototype/design files.

### Example — confirmed working

**Make file tested:** `https://www.figma.com/make/2V6S9XdKyPL6WfTDD63yiN/Recreate-Editable-Figma-Mock`

Steps that worked:
1. `get_design_context(fileKey="2V6S9XdKyPL6WfTDD63yiN", nodeId="0:1")` — returned source file URIs
2. `ReadMcpResourceTool(server="figma", uri="file://figma/make/source/2V6S9XdKyPL6WfTDD63yiN/src/App.tsx")` — returned full 1200+ line React/TypeScript source
3. Derived 35 screens from the `Screen` type union and `setCurrentScreen` calls
4. Generated `UpgradeReadiness_UserFlowDocument.pdf` (37 pages) and `UpgradeReadiness_TestSpecification.xlsx` successfully

**UI framework found in that Make file:** `@ui5/webcomponents-react` (Button, CheckBox, FlexBox,
Input, MessageStrip, ObjectStatus). Playwright selectors for UI5 Web Components should use
`getByRole()` with accessible names, e.g. `page.getByRole('button', { name: 'Auto Fix' })`.
