# The permission prompt tool

This file describes the **steps followed** to add Concept 37 (**the permission prompt tool**) to the Claude Agent SDK
Lab: what was read, what was probed, what was decided, how it was tested, and what the tests changed. To learn the
concept itself, read [Tab37-Permission-prompt-tool.md](Tab37-Permission-prompt-tool.md) (the same text is in
`readme.md`).

| Concept | Topic | Routes | Explanation |
|---|---|---|---|
| 37 | The permission prompt tool: `permissionPromptToolName`, the call and its JSON answer, `canUseTool` as `--permission-prompt-tool stdio`, an in-process gate that asks the user, an external policy gate (stdio MCP server), who is asked and when, `updatedInput`, `updatedPermissions`, `interrupt`, failures | `/api/c37/dry`, `/run`, `/decide`, `/gate-log`, `/who`, `/answers`, `/failures`, `/code` | [Tab37-Permission-prompt-tool.md](Tab37-Permission-prompt-tool.md) |

## How to run it

```powershell
npm run dev        # server on http://localhost:3001, web on the Vite port
```

`node_modules` was copied from sample36, so `npm install` is not needed. Open the **37. Permission prompt tool** tab.
`ANTHROPIC_API_KEY` must be in `.env`. Start it from a normal terminal, not from inside Claude Code (see Tab16). A
scenario costs between $0 (scenario 1) and about $0.06 (scenarios 4 and 5).

> **Only one sample can run at a time.** Every sample's server uses port **3001**. Stop the other samples'
> `npm run dev` first.

## Step 1: Choose the feature

The request said "implement the following feature sample" with no feature text, as in samples 28 to 36. Sample 36's
Step 1 had listed the `Options` fields of `sdk.d.ts` (`0.3.281`) used nowhere in `server/` or `src/`. After Concept 36,
eight were left: `betas`, `maxThinkingTokens`, `onUserDialog`, `permissionPromptToolName`, `projectConfigRoot`,
`promptSuggestions`, `resumeDropsTurn` and `supportedDialogKinds`. **`permissionPromptToolName`** was chosen:

- It completes the permission story of Concepts 4, 20 and 31. The SDK's code shows that `canUseTool` itself is sent
  as `--permission-prompt-tool stdio`, and that the two options throw when combined.
- It has a real use: an approval service outside the app, shared by many agents.
- It can be shown with real runs and cheap tables. The others are deprecated (`maxThinkingTokens`), model-specific
  (`betas`), or need a flow the lab cannot trigger on demand (`onUserDialog`, `resumeDropsTurn`).

## Step 2: Read what the course already said

| Read | To learn |
|---|---|
| `Build-steps.md`, `36-process-spawning.ts`, `Concept36ProcessSpawning.tsx`, `Tab36-Process-spawning.md` | The latest style: strict zod bodies, SSE events, tables of cases run in parallel, `#region` + `/code`, a summary table, the dry run with a fake process |
| `31-ask-user-question.ts`, `04-permissions.ts`, `Tab4-Permissions.md` | The "waiting room" for a human answer (a `pending` map, a timeout, abort), and what `canUseTool` and `permissionPrompts` already taught |
| `mcp-servers/notes-server.ts` | How the course writes a stdio MCP server (never write to stdout, config through `env`, POST a log to the lab) |
| `server/index.ts`, `src/App.tsx`, `src/lib/sse.ts`, `src/styles.css` | Mounting, the tab list, SSE, the CSS classes to reuse |

## Step 3: Read the types and the SDK's code

| Found | Used for |
|---|---|
| `Options.permissionPromptToolName` ("MCP tool name to use for permission prompts") and `permissionPrompts: 'host' \| 'none'` ("this process, through `canUseTool` or `permissionPromptToolName`") | Parts A and D |
| In `sdk.mjs`: `if (canUseTool) { if (permissionPromptToolName) throw …; args.push("--permission-prompt-tool", "stdio") } else if (permissionPromptToolName) args.push("--permission-prompt-tool", name)` | The dry run and the first failure row |
| `PermissionResult` (`allow` + `updatedInput` + `updatedPermissions`, `deny` + `message` + `interrupt`), `PermissionUpdate` (`addRules`, `destination`) | Part E |

The types say nothing about **what the tool receives or must return**. That is what the probes had to find out.

## Step 4: Probe before designing

Scratchpad scripts (in a temporary `.probe/` folder) ran real sessions with Haiku 4.5:

| Probe | Result | Decision |
|---|---|---|
| An SDK MCP `approve` tool + a stdio tap (Concept 36) | Claude Code sends `control_request/mcp_message` `tools/call approve` with `{ tool_name, input, tool_use_id }`; a JSON `PermissionResult` as text works; `system/init` does not list the tool | Step 1 of the doc, Part B |
| 15 cases in parallel | deny → `is_error` + `permission_denials`; `updatedInput` really changes what runs; text that is not JSON → denial "invalid permission result"; a throw or `isError` → a tool error (not a denial), the model retries; a missing tool → exit 1 at the first prompt, and the real tool is then visible to the model; `interrupt` → `error_during_execution`; `allowedTools`, `acceptEdits` and `permissionPrompts: 'none'` decide before the gate; a session rule stops later prompts; a stdio gate works with one pid for the whole session | Parts D, E, F |
| A 70 s gate, hooks, a subagent, a `localSettings` rule, extra input fields | No timeout; a hook's `deny`/`allow` decides first, its `ask` reaches the gate even in `acceptEdits`; `.claude/settings.local.json` written; no extra fields sent. The subagent case was flaky (a background agent, interrupted calls), so it was **left out** | Part B's own 2-minute limit, the hook rows |
| `echo hi` with Bash | Never reached the gate (a read-only command) | The lab's prompts use `mkdir` and `rm` |

## Step 5: Design the concept

- **Two gates.** An **in-process** gate (an SDK MCP tool in the server) that waits for the browser, so every call and
  answer can be seen and chosen. And an **external** gate, `37-policy-gate.mjs`: a separate process that Claude Code
  starts, with a JSON policy from its environment. It reports each decision to `POST /gate-log` with a per-run token.
- **The same two steps** (write `a.txt`, then `mkdir out`) for the tables, so that only the setup or the answer
  changes between rows. `a.txt` and the run folder are read after each row, to compare what ran with what the model
  said.
- **Routes**: `POST /dry`, `POST /run` (SSE, `you` or `policy`), `POST /decide`, `POST /gate-log`, `POST /who`,
  `POST /answers` and `POST /failures` (SSE, one row each), `GET /code` (the regions + the gate's source + the default
  policy).
- **The tab**: A (the dry run), B (you decide), C (the policy gate, with the policy editable), D (who is asked),
  E (the answers), F (failures), G (code), H (a summary table).

## Step 6: Implement it

| File | What was done |
|---|---|
| `server/concepts/37-permission-prompt-tool.ts` | New: `base()`, `sdkGate()`, `askTheBrowser()` and `/decide`, `policyGate()` and `/gate-log`, `relay()`, `argsOf()` (the dry run), `runRow()` and the `WHO`, `ANSWERS`, `FAILURES` rows, the routes |
| `server/concepts/37-policy-gate.mjs` | New: the policy gate |
| `server/index.ts` | Mounted on `/api/c37` |
| `src/concepts/Concept37PermissionPromptTool.tsx` | New: `Decision` (the card), `Timeline`, `DryTable`, `RowsTable`, Parts A to H |
| `src/App.tsx`, `src/styles.css` | Tab; `.tag-gate`, `.tool-call.gate`, `.gate-card`, `.policy-edit` |
| `.gitignore`, `Tab1-query().md` | `gate-lab/`, table row and project tree |

`npx tsc --noEmit -p .` passed.

## Step 7: Test the routes

Port 3001 was left free for the other samples. A scratchpad script (`serve.ts`) mounted **only** the Concept 37 router
on port **3137**, with `.env` loaded and the `CLAUDE*` variables removed. `drive.mjs` posted each body and printed each
SSE event. `human.mjs` ran `/run` with the gate `you` and answered each card through `/decide`.

| Test | Result |
|---|---|
| `/run` with `gate: "mars"`, a policy with `default: "maybe"` and the regex `(`, `/decide` `edit` without `input`, `/decide` for an unknown id, a forged `/gate-log` | Refused with a clear message (400 / 404 / 403), nothing started |
| `/dry` | The six rows of the doc's Step 2 |
| `/run` `you` (edit, remember, deny) and `policy` | The timelines of the doc's Steps 3 and 4 |
| `/who`, `/answers`, `/failures` | The rows of the doc's Steps 5, 6 and 7 |

What the tests changed:

- **Every row with the in-process gate failed** with *MCP tool mcp__gate__approve … not found. Available MCP tools:
  none*, although `system/init` said `gate: connected`. A `debugFile` showed `tools/list failed (Cannot read properties
  of undefined (reading 'push'))`, and a script listing the tools in-process reproduced it: the SDK cannot list a tool
  whose schema has `z.record(…)`. The probes had used `z.any()`. The gate now uses `z.looseObject({})`, and the trap is
  a failure row and Step 9 of the doc.
- **"Allow and remember" added a rule for the whole Bash tool**, and it silently allowed the later `rm haiku.txt`. It
  now adds `Bash(<first word>:*)`, and the `rm` is asked (and denied).
- **The "external gate does not start" note claimed** that `system/init` reports the failure. That was then checked
  (`gate: failed`), and the note now prints the real status.
- Rows whose Claude Code exits have **no result, so no cost**. The table says "cost unknown" instead of $0.

## Step 8: Run it in the real app

Vite ran on port **5199** with a scratchpad config whose proxy pointed to the 3137 server. Headless Edge was driven
through the DevTools protocol (`shot.mjs`, profile in `%TEMP%\c37-edge`), with real runs: 1, 2 (Allow, Allow and
remember, Deny with a message typed in the card), 3, 4, 5, 6.

| Check | Page |
|---|---|
| Open tab 37 | 10 code buttons, the default policy in the editor |
| 1 | The six-row flag table, the `both` row in red |
| 2 | The decision card under the `approve()` row, the `tool_use_id` linked to the `tool_use`; the run folder: `archive/, backup/, haiku.txt` |
| 3 | 5 policy gate rows with the gate's pid; the run folder: `notes.txt, out/` |
| 4, 5, 6 | 7, 7 and 6 rows |
| Console | No error |

What the browser run changed:

- **The timeline of run 2 was shown under Part C**, far from the button and the prompt of Part B. The timeline now
  shows under the part whose button started it.

All test processes were stopped, and `gate-lab/`, `.probe/` and the Edge profile were deleted.

Costs: under $1 for all the probes, route tests and the browser test.

## Files added or changed

| File | Change |
|---|---|
| `server/concepts/37-permission-prompt-tool.ts` | New: the Concept 37 routes, the gates, the tables of cases |
| `server/concepts/37-policy-gate.mjs` | New: the external policy gate (a stdio MCP server) |
| `server/index.ts` | Mounts `/api/c37` |
| `src/concepts/Concept37PermissionPromptTool.tsx` | New: the Permission prompt tool tab |
| `src/App.tsx` | Tab |
| `src/styles.css` | Gate rows, decision card, policy editor |
| `.gitignore` | `gate-lab/` |
| `Tab1-query().md` | Adds Concept 37 to the table and the tree |
| `Tab37-Permission-prompt-tool.md` | Explanation of the concept |
| `readme.md` | Same content as `Tab37-Permission-prompt-tool.md` |
| `Build-steps.md` | This file |
