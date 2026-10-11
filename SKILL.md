---
name: browser-skill
description: |
  Use when the user asks to automate their logged-in Chromium browser: visit
  and read pages, fill forms, scrape data, click through flows, regression-test
  a PR's UI, validate a deployed page, or operate a tab they identify. Requires
  the bsk CLI and browser extension.
---

# browser-skill

Use `bsk` to work in an **Agent Window** with the user's existing logins. User tabs
require explicit borrowing. This skill does not install the extension or handle
advice-only tasks. Never extract credentials, cookies, tokens, or other secrets.

## Before starting a session

For remote setup or pairing, follow the [remote guide](https://github.com/fish0710/BrowserSkill-internal/blob/fork-release/docs/remote-extension-connection.md).

Local commands normally auto-start the daemon. If the host terminates background
children after each shell call, including on Windows, complete these steps first:

1. Reuse the host daemon's existing `BSK_HOME` (or its default if unset). Set
   `BSK_AUTO_START=0` and run `bsk status --json`. Reuse a working daemon; an empty
   `browsers` list means the extension still needs connecting. Permission errors,
   timeouts or invalid replies do not prove the daemon is absent.
2. Only if the check reports a missing daemon and no host task is already starting
   it, run `bsk daemon start --foreground` with the same `BSK_HOME` in the host's
   approved persistent background task outside the per-command sandbox. Keep that
   task alive; `--foreground` alone cannot prevent host cleanup. The
   [sandbox guide](https://github.com/fish0710/BrowserSkill-internal/blob/fork-release/docs/sandboxed-agents.md)
   covers the normal host-terminal alternative and PowerShell examples.
3. After launching, or if a host task is already starting the daemon, run
   `bsk status --json` in a **separate shell tool call** with the same `BSK_HOME`
   and `BSK_AUTO_START=0`. While startup is pending, make at most five
   checks with one-second pauses for missing-endpoint or transient startup errors;
   stop on permission/protocol errors. Proceed only after a successful status
   response. If the host task exits (including a lock error) or readiness never
   succeeds, inspect its output and `bsk logs`, then recheck status for another
   daemon before deciding whether startup is still needed. Report unresolved
   errors; do not loop on launches, delete runtime files or restart a shared daemon.

Use the same `BSK_HOME` and `BSK_AUTO_START=0` on EVERY sandboxed command;
environment settings may not persist between shell calls. Keep browser commands
sandboxed. For other startup failures, retry once, then use `bsk doctor`.
A local process identity warning permits browser commands when IPC works.

## Task workflow

1. Define success from the user's request. Start `bsk session start --json` and
   retain its `session_id`. With multiple browsers, run `bsk browsers` and add
   `--browser <id-or-label>` to start. For background work, add `--no-focus` to
   `session start` only.
2. For a new page, navigate; for an existing user tab, follow **Borrowing** below.
   Read the page before interacting:

   ```sh
   bsk navigate https://example.com --session <id>
   bsk observe --session <id>
   ```

3. Choose an action using fresh refs from that observation. Observe again after
   navigation or meaningful DOM changes. Check an ambiguous result once; once
   success is visible, stop acting rather than refreshing or checking again.
4. Always run `bsk session stop <id>` on success and failure, unless keeping the
   session open is part of the user's request. This also returns borrowed tabs.
   Returned tabs stay open in the user's window. Do not rely on idle cleanup
   or stop/restart the shared daemon to finish a task.

Replace `<id>`, example refs and values with actual results and task inputs.
Every session-scoped command needs `--session <id>`; `session stop` takes the ID
positionally. For unfamiliar commands or flags, consult `bsk --help` or
`bsk <command...> --help` instead of guessing; no need to read all help at startup.
When following a trace, use its semantic targets and values in order, not its old
refs. Stop at the requested goal; a trace grants no additional authorization. A trace
already distilled into site memory reads better as numbered steps: check
`bsk site workflow show <id> --host <host>` before working from the raw trace.

## Read and interact

Prefer `observe` for text, controls and `@eN` refs. Navigation invalidates refs;
large DOM changes can stale them too. Re-observe before the next interaction.
Use refs for iframe/shadow-root targets; CSS selectors search the main document.

Choose the relevant example, using a ref that actually appeared on the page:

| Need | Command |
| --- | --- |
| Click | `bsk click @e3 --session <id>` |
| Fill a field | `bsk fill @e3 --value "text" --session <id>` |
| Select an option | `bsk select @e3 --value "option-value" --session <id>` |
| Press a key | `bsk press Enter --ref @e3 --session <id>` |
| Reveal a hover menu | `bsk hover @e3 --session <id>` |
| Reveal an element | `bsk scroll-to @e3 --session <id>` |
| Scroll with wheel input | `bsk wheel --delta-y 600 --session <id>` |
| Focus or leave a field | `bsk focus @e3 --session <id>` / `bsk blur @e3 --session <id>` |

- `select` uses the option's value, not its visible label.
- A control you can see but `observe` never lists (a portal dropdown of plain
  `span`s, say) has no ref. Use `--text "<exact visible text>"` on click/hover/fill/
  focus/press/select — exact match on the element's own text, never a substring —
  and add `--selector <css>` to scope it when the same text appears elsewhere:
  `bsk click --selector .dropdown-item --text "后端" --session <id>`. Ambiguity is
  an error, not a guess; `get-html` shows the classes to scope by. A bare
  `--selector` must match exactly one element too. To act inside one row of a
  list, confine the lookup to that row: `bsk click --selector span.status --text
  "待处理" --within-selector tr.ant-table-row --within-attr data-row-key
  --within-value <rowId> --session <id>` (`--within-text-of <css>` compares a
  cell's text instead of an attribute; needs extension protocol 1.7).
- Hover markers such as `[hover first: Shoes | Bags]`, `[has-submenu]`, or
  `[expanded]` identify triggers. Hover the trigger, observe, then use the revealed
  item's ref. Listed labels are not refs; do not click the trigger unless its own
  action is wanted. If an expected control is missing and no marker identifies a
  trigger, try `observe --probe-hover` once. It touches the live page and costs
  seconds; use targeted hover once the trigger is known.
- `scroll-to` returns ancestor-clipped bounds in top-level viewport CSS pixels.
  Partial visibility suffices; hidden/fully clipped targets fail. It does not test
  occlusion. `wheel` sends signed deltas (at least one nonzero), not a guaranteed
  scroll distance. An optional target is scrolled into view first; without one,
  input lands at the viewport centre. Observe to check the page's response.

Use `snapshot` for a static accessibility tree, `get-html` for exact markup or
hidden metadata, and `screenshot` for visual content or requested visual evidence.
Do not start with HTML/images just to find ordinary controls; obtain fresh refs
before interacting with controls found that way.

### Large observations

There is no default token cap. With `observe --max-tokens <n>`, follow a returned
`next_cursor`/`@more` when relevant content remains:

```sh
bsk observe --cursor <token> --session <id>
```

Each page replaces the ref map: use its refs before continuing and never reuse
refs from earlier pages. Continuation reads the same capture, without refreshing
or hovering; do not combine it with depth changes or hover probing. New observe/
snapshot or changed page identity invalidates continuation; then observe afresh.

## Borrowing and browser settings

List before borrowing, and return the tab as soon as the relevant step ends:

```sh
bsk tab list --scope user --session <id>
bsk tab borrow <tab-id> --session <id>
bsk tab return <tab-id> --session <id>
```

Never invent tab IDs or keep a user tab across unrelated work. Do not repeat
pending, denied or timed-out borrows. For `borrow_outcome_unknown`, inspect tab/
session state first: the tab may already have moved. Do not bypass an outcome
through another browser backend. `tab borrow --timeout 120s` changes only the
confirmation wait (default 60s); custom waits require daemon and extension protocol 1.2+.

The extension's saved Automation settings control borrow confirmation and human
help independently; both default on and apply to existing sessions too. Read
`interaction` in `session start --json` or `session list --json` when needed.
Deprecated `--unattended`, `--no-confirm`, and `BSK_REQUEST_HELP=off` cannot override
these settings. Never change browser storage/settings to bypass them. Human-help
availability does not require permission for every action or grant extra authority.
`request-help` requires daemon protocol 1.3; update CLI, daemon and extension for
full settings support. A feature's version error does not disable other operations.

Remote content reads/actions require task-created or borrowed tabs. Page-opened
popups gain no control automatically; an unowned tab inside the Agent Window
needs the user to move it to a user window before borrowing. Remote upload/download
are unsupported; screenshots work.

## Human steps and recovery

With help enabled, request help for login, CAPTCHA, OTP, payment confirmation,
consent, or after two attempts make no progress:

```sh
bsk request-help --session <id> --prompt "Please complete sign-in" --target @e3
```

Use a precise prompt and fresh targets; omit `--target` when no control fits.
Use completion criteria only for a clear, stable success signal.

| Result | Next step |
| --- | --- |
| Help `continued` / `completed` | Observe again, then resume with fresh refs. |
| Help `cancelled` / `timed_out` | Respect rejection or the blocker; do not repeat the request. |
| Help `disabled` | No human action was confirmed. Re-observe and follow the disabled-help rules below. |
| Stale ref | Observe and retry the intended action once. |
| Unknown tab/session | List current tabs/sessions; never guess IDs or use another task's session. |
| Timeout or unknown effect | Inspect current state before retrying; the action may already have happened. |
| `fill_value_mismatch` | Read the field: formatting may still satisfy the request. Correct only a remaining difference; no blind refill or immediate handoff. |
| Unsupported operation | Use available capabilities; suggest updating only if the missing feature is needed. |

Navigation alone (including deprecated help outcome `navigated`) is not completion.
For other errors, follow the returned hint and inspect the current state.

**Help disabled:** do not request help or re-enable it. Use existing login state,
authorized inputs and viable alternatives; disabling help adds no permission and
does not remove borrow confirmation or host restrictions. Where authorized, a
vision-capable model may attempt graphical verification. Phone-only QR scans,
face verification, missing SMS codes or image-only tasks for a text-only model
may remain blocked. Report a specific blocker only when inputs/capabilities are
missing or viable approaches are exhausted; continue independent work. Do not loop
on identical failures, repeat unknown effects or switch backends to bypass limits.
On an unrecoverable failure, report the blocker and stop the owned session.

### User takeover

The user can press "Take over" in the Agent Window at any time. The session is then
held (`control=user`) and every browser-input call is rejected with `tool dispatch
rejected: the user has taken over this session (control=user)`. That means **stop
acting**: do not retry or route around it. Run `bsk session wait-control --session <id>`;
it blocks until control returns and prints the user's `note`. `bsk session status
--session <id>` shows the state without blocking. Re-observe before continuing: the
user may have navigated, filled or submitted something.

## Screenshots and Canvas

```sh
bsk screenshot --session <id> --out viewport.png
bsk screenshot --session <id> --ref @e3 --out element.png --json
bsk screenshot --session <id> --full-page --out page.png
bsk screenshot --session <id> --full-page --scope current --out loaded.png
```

Screenshots return a local PNG path; view the image to interpret it. `--out`
replaces an existing file; omitting it uses a temporary path. `--json` includes
dimensions and byte size. `--ref` and `--full-page` cannot be combined.

Full-page mode scrolls an ordinary webpage and restores its position/styles.
The default `--scope follow` follows appended content. Use `--scope current` when
capturing the currently loaded range is requested: it stops at the initial document
height, even if a loading indicator remains. Later content below that boundary is
excluded; report this range rather than claiming all feed entries were loaded.
Use a selected, session-controlled tab and stable viewport; `--tab-id` targets a
tab without selecting it. Internal browser pages, the Web Store, nested scrolling
panels and virtualized lists are unsupported. Capture/encoding defaults to 2m;
`--timeout 5m` extends it only in full-page mode. Allow the shell enough time for
capture plus transfer. Respect cancellation; do not blindly retry endless pages
or substitute a viewport image when an older extension rejects full-page capture.
Use matching CLI/extension builds. Ctrl-C cancels; failed full-page captures save
no partial image. A `loading_stalled` error means the bottom kept a loading
indicator without height growth for 30s; do not simply increase the deadline.
Choose `current` only when that range satisfies the request. Keep the capture tab
visible: `page_hidden` is an environment interruption, while `user_cancelled`
means user input stopped capture. For other failures follow the returned reason
and hint; do not work around them by editing the page or stitching screenshots.

For `@eN canvas [visual:screenshot]`, observe returns text, not pixels. Screenshot
that ref when its contents matter; never infer Canvas controls or names from
nearby labels. If images cannot be received/understood, explain the limitation,
ask for an image-capable model when needed, and continue with available semantics.

To click a point seen in a Canvas image, retain that screenshot's `capture_id`:

```sh
bsk click @e3 --capture <capture-id> --image-x <x> --image-y <y> --session <id>
```

Use ORIGINAL PNG coordinates and dimensions, not resized display/viewport pixels.
Captures are single-use, expire after 2m, and are invalidated by ref replacement
(observe/snapshot/continuation) or a newer screenshot of that ref. With
`capture_unavailable`, the image is view-only: observe and screenshot again before
clicking. Counts 1/2, buttons and modifiers work; Canvas fill, IME, drag, hover
and HTML extraction do not. Repainting is allowed; changed identity/geometry/hit
targets are rejected. Verify the result, using DOM refs for revealed controls;
inspect `effect_state=unknown` before retrying with a new capture.

## Files and other tools

```sh
bsk upload @e3 --file ./report.pdf --session <id>
bsk download @e3 --out ./report.pdf --session <id>
```

Upload discloses the file to the site; download accepts site-controlled bytes.
Use agent-local paths, not browser-internal staging paths.

- Default upload clicks an upload button/label and intercepts its file chooser.
- If `reason=file_input_not_activated` and `effect_state=none`, re-observe. Try
  `--mode drop` once only on a clear attachment target such as a drop zone or
  composer, never whitespace or an ambiguous container. Otherwise follow the
  human-help rules. There is no automatic fallback between mechanisms.
- Never retry or switch upload modes for `effect_state=unknown` or `committed`.
  A successful drop proves dispatch, not site acceptance; observe the attachment.
- Download refuses overwrite by default; add `--overwrite` only when replacement
  is intended. Consult each command's help for other flags.

Use `console` / `network` for bounded read-only diagnostics; follow returned
sequence cursors. `emulate --device iphone-14` affects one tab; `--off` restores it.
`evaluate` is a last resort: inspect JSON `.ok`, since a script exception can have
CLI exit code 0. Never evaluate secrets. `record start` captures user actions, or
with `--detach` your own; read its help first and never record banking, SSO or
password-manager pages.
Use `bsk --help` to find navigation/history, tab, wait and window commands.

## Site memory

`bsk site` keeps local, per-host notes so an explored flow is not re-explored: a
constrained `SITE.md`, workflows derived from a recording, and candidate observations
awaiting evidence. It is private to this machine, needs no daemon, and never stores
credentials or recorded input values.

```sh
bsk site context --host <host> --task <task-id>
bsk site workflow show <id> --host <host>
bsk site workflow save --from ./rec/trace.json --id <id> --task <task-id>
bsk site workflow verify <id> --host <host> --pass
bsk site candidate add --host <host> --kind better_path --claim "<one sentence>"
bsk site checkpoint --host <host> --task <task-id> --reason direct_correction
```

- **Read before acting.** When a task names a site, run `bsk site context` first and
  follow what it already knows.
- **Run the script before driving by hand.** When `site context` lists a verified
  workflow whose purpose matches the task, `bsk site workflow run` it with the task's
  values as `--param` — do not re-observe and re-click the same steps yourself. Only
  when the run stops with `workflow_step_failed` do you take over, and only from that
  step; then fix the anchor of the failing step (role/name, `text`, or `selector`) so
  the next run passes.
- **One window per task.** Every session opens its own Agent Window, so a run, a
  re-run and a check done in separate sessions flash three windows at the user. Run
  with `--keep-session` (or inside a session you already started, with `--session`),
  observe the result in that same session, and stop it once at the end:
  `bsk site workflow run <id> --host <host> --param … --keep-session` → the last line
  names the session → `bsk observe --session <id>` → `bsk session stop <id>`. Never
  start a second session just to look at what the first one did. A run inside a kept
  session also ends faster: the last step's navigation grace drops from 1200ms to 500ms.
- **`verify --pass` attests a `workflow run`, never a manual run.** Clearing NEEDS
  REVIEW by hand-driving the same page proves nothing about the anchors.
- **Never explore to learn.** Record only what the task itself revealed; no extra
  pages or detours to make memory "more complete".
- **Learning never fails the task.** These commands are advisory: on failure report
  one line and continue the user's goal. Do not retry or let it block the task.

To leave memory behind for the task you are already doing, record your own run:

```sh
bsk record start --detach --url <start-url> --output ./rec --json   # prints session_id
bsk observe --session <id>                                          # do the real task
bsk fill @e4 --value "<text>" --session <id>
bsk press Enter --session <id>
bsk record stop --output ./rec                                      # exports trace.json
bsk site workflow save --from ./rec/trace.json --id <id> --task <task-id>
bsk site checkpoint --host <host> --task <task-id> --reason direct_correction
```

`workflow run` replays a stored workflow without an LLM. A step with a `selector` is
tried first, directly in the live DOM (`selector`+`text` scopes the selector to the one
element with that exact text) — one query, no page observe. Only when that finds nothing
or more than one element does it observe the page and match `role` + `name` (+ `ctx`) to
exactly one `@eN`; a step with only `text` goes `role` + `name` first and `text` as the
fallback, because bare text lands on the innermost matching element (a label, not the
control around it). The recorder writes `text`/`selector` for every click it
can, and an attribute selector (`input[placeholder="…"]`, `[name]`, `[aria-label]`, a
stable `#id`) for form fields; you may also write them by hand into the workflow JSON. It
is the one `bsk site` command that needs the daemon and a browser.

- **Keep every step cheap.** A step with a `selector` is located in milliseconds; a step
  with only `role`/`name` observes the whole page first, which costs seconds on a large
  page and is what makes a replay feel jerky. After `workflow save`, read `workflow show`:
  give any click/fill step that has no `selector` one by hand (`button.ant-btn-primary`
  plus `text: "创建"`, `input[placeholder="…"]`) — a selector that is unique on the page,
  never an `@eN`. Keep `role`/`name` as well; they are the fallback if the selector breaks.
- **One row of a list: `within`, never a parameter in the selector.** When the same
  control repeats per row (a status cell, an "edit" link), a selector + text that was
  unique while recording turns ambiguous once the list grows. Pin the row by its
  identity and pass the identity as a parameter:

  ```json
  "params": [{ "name": "taskId", "required": true, "pattern": "[A-Z]{4}[0-9]{6}" }],
  "steps": [
    { "n": 1, "op": "navigate",
      "toTemplate": "https://<host>/list?detailId={{taskId}}&tab=child" },
    { "n": 2, "op": "click", "target": {
        "selector": "span.list-scrum-status", "text": "待处理",
        "within": { "selector": "tr.ant-table-row",
                    "key": { "from": "taskId", "attr": "data-row-key" } } } }
  ]
  ```

  `within.key` takes exactly one of `from` (a parameter) / `value` (a constant) and
  exactly one of `attr` (an attribute of the row) / `textOf` (a css for the cell whose
  whole text is the id); the target still needs `selector` or `text`. Such a step is
  located only inside that one row — never page-wide — and fails with `ambiguous` /
  `not_found` rather than acting on another row; keeping the status text (`待处理`)
  makes it a state check as well. `toTemplate` (instead of `to`) substitutes
  `{{param}}` URL-encoded; the result must stay on the workflow's host. `{{…}}`
  anywhere else is rejected. A parameter used by `within` or `toTemplate` must be
  `required`, not `secret`, passed with `--param` (not `--param-env`), non-empty, ASCII,
  without surrounding whitespace, and match its `pattern` (whole value) — otherwise
  the run stops with `param_invalid` before touching the browser. These workflows,
  and any step that locates by `selector` alone, need extension protocol 1.7; an
  older extension fails the run up front with `extension_too_old` (`bsk update`,
  then reload the extension).
- **Read the timings before tuning.** `--report ./run-report.json` lists per step
  `observeMs`, `actMs`, `settleMs` and `settleReached`. A step with `observeMs` in the
  seconds is missing a selector; `settleReached: "none"` on every click means no step
  navigates and the run is already as tight as it gets.

```sh
bsk site workflow run <id> --host <host> --param title="<text>" --param-env token=API_TOKEN \
  --step-timeout 8s --report ./run-report.json
bsk site workflow run --from ./derived/form.json --param title="<text>"   # or a file
```

Pass either `<id> --host` or `--from <workflow.json>`, never both. A `--from` file is
validated exactly like a stored workflow (schema, no `@eN` refs, refused hosts) and reports
the host it declares itself.

Secret parameters accept only `--param-env`, and anything read that way prints as `***`.
When a step cannot be matched uniquely the run stops with `workflow_step_failed`; `data`
carries the step number, a `reason` (`not_found` / `ambiguous` / `tool_error` /
`postcondition_failed` / `needs_review`) and the candidate lines. Take over by hand from
that step — do not re-run hoping for a different match. A `not_found` whose `observeError`
is set means the observation itself failed (a disconnected daemon, a page mid-navigation),
not that the element is gone; the report also keeps the failing step's last observation
under `observation` so you can read the page as it was.

`--detach` returns once recording is armed, so the session accepts your commands;
without it `record start` holds the session and every command returns `session_busy`.
`record stop` also ends that session. Reusing a workflow needs no recording: read it,
run it, then `workflow verify --pass`, which clears NEEDS REVIEW.

`workflow save` writes into a per-task draft; `checkpoint` publishes it. Revisions are
per host. A `conflict` result means another writer committed first: re-run
`bsk site context`, which re-seeds the draft's `SITE.md` from the published revision
(your draft edits are gone and must be replayed; staged workflows are kept), then
checkpoint again. Retry a conflict only once. Recorded `fill` values become named
parameters and are never stored; pass `--inline-values` only when the values belong
to the flow, not to the person who recorded it. `<select>` option values are site
constants and stay inline. Banking, SSO and password-manager hosts are refused.
