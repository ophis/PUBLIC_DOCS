# TASK-166 spec: the tui runner waits through background work, gives up after 2 h quiet

## Problem

`drive.Tui` treats every turn end (`stop`) without an outcome as stopping: one NUDGE, then `STOP_LIMIT` (3) more stops → it gives up. A run waiting on background work (sub-agents, shells, monitors, workflows) wakes and ends a turn each time one finishes, so TASK-165's engineer was abandoned at 21:36 and its 22:00 outcome was never read.

Claude Code's Stop hook gets the hook input JSON on stdin; `background_tasks` lists the in-flight tasks, present (possibly `[]`) when the task registry is reachable, omitted otherwise (https://code.claude.com/docs/en/hooks.md › Stop input).

## Behavior

### 1. `core/src/report.py stop [--pending KEY]`

- With `--pending KEY`: read stdin. If it parses as a JSON object whose `KEY` value is a list → append `{"kind": "stop", "pending": <len>}`. Otherwise (stdin empty, invalid JSON, undecodable, not an object, `KEY` missing or not a list) → append `{"kind": "stop"}`, exit 0, as today.
- Without `--pending`: stdin is not read; append `{"kind": "stop"}` (today's behavior, so a hand-run `report.py stop` never blocks on a terminal).
- stdout stays `report.py: stop reported`.
- `KEY` is opaque to report.py: no Claude field name in it.

### 2. Claude client (`core/src/clients/claude.py`)

- The Stop hook (still in `--settings`) runs `<report_command> stop --pending background_tasks`.
- `background_tasks` appears only in `clients/claude.py` and tests; core's other modules and docs outside the client say "pending background work".

### 3. Channel → event

- `clients.Event` gains `pending: int = 0` (the in-flight background work count a `stop` carries).
- `drive.report_event`: `{"kind": "stop", "pending": N}` with `N` an `int` (not `bool`) ≥ 0 → `Event("stop", pending=N)`; any other `pending` (missing, negative, non-int) → `Event("stop")`.

### 4. `drive.Tui`

- `stop` with `pending > 0`: no nudge, not counted toward `STOP_LIMIT`, `nudged` unchanged.
- `stop` with `pending` 0 (or missing): today's rule — first one gets NUDGE; after it, `STOP_LIMIT` counted stops → give up; a progress report resets the count.
- **Quiet limit:** new constant `WAIT_LIMIT = 2 * 60 * 60` (seconds). The clock (`time.monotonic()`) starts when `begin` has started the session (the run's start) and restarts at every progress report. When no outcome has arrived and more than `WAIT_LIMIT` seconds have passed, the runner gives up exactly as at `STOP_LIMIT`: `gave_up`, `done` → True, session left open, stderr line `drive.py: no outcome <WAIT_LIMIT as hours> h after the last progress report; session <name> left open: tmux attach -t '=<name>'` (wording may differ but it names the session and the attach command); `drive.start` then returns `Result(0, None, "the run returned no outcome")`, `drive.py` exits 1.
- Checked once per poll (in `done`, after `outcome_arrived`, so an outcome in the same poll still wins), independent of stop events, so it also covers a client without a stop hook.
- Both give-up paths go through one helper that sets `gave_up` and prints the attach line, the reason the only difference.
- Stop events after an outcome or after giving up change nothing (as today).

### 5. Docs

Update every place this makes wrong, briefly:
- `README.md:79` (the `Stop` hook paragraph: pending stops, the 2 h quiet limit; drop "There is no wait timeout").
- `core/src/drive.py` module docstring (tui: done once the outcome arrives or it gives up), the `STOP_LIMIT` comment if needed, the `Tui` docstring.
- `core/src/report.py` docstring (`stop [--pending KEY]`).
- `core/src/clients/base.py` `Event` docstring (`pending`, meaningful on `stop` only).
- `core/CLAUDE.md` › Add a client › 3 (stop may carry `pending`: `stop --pending KEY` counts the list at top-level `KEY` of the JSON on stdin, so a client whose hook input has another shape needs another option; "it has no wait timeout" is now false).
- Any other doc found stating the old rules.

## Tests (written first, `core/src/tests/drive_test.py`)

- report.py: `stop --pending background_tasks` with stdin `{"background_tasks": [{…},{…}]}` → `{"kind":"stop","pending":2}`; `[]` → `pending: 0`; empty stdin, invalid JSON, a JSON list, key missing, key not a list → `{"kind":"stop"}` and exit 0; `stop` without the flag reads no stdin (patch `sys.stdin` with an object whose `read` fails the test).
- `report_event`: pending int → `Event("stop", pending=N)`; `true`, `-1`, `"2"` → `Event("stop")`.
- Claude client: the hook command is `… report.py --to <workdir>/.report.jsonl stop --pending background_tasks`.
- Tui: a run of more than `1 + STOP_LIMIT` pending stops, then an outcome → done, no `send`; pending stops interleaved with pending-0 stops count only the pending-0 ones (nudge on the first pending-0 stop, give up after `STOP_LIMIT` more); the quiet limit gives up after `WAIT_LIMIT` with no progress (prints the attach line, leaves the session, `main` exits 1); a progress report restarts the clock (no give-up at `WAIT_LIMIT` from start when progress came later); an outcome arriving after `WAIT_LIMIT` elapsed with no poll in between still counts (done on outcome).
- Time is faked by patching `time.monotonic` (stdlib) — code under test unmodified.
- Grep check: `background_tasks` occurs in `core/src/` only in `clients/claude.py` and tests.

Verify: the three suites in `CLAUDE.md` pass:
```
python3 -m unittest discover -s orchestrator/src/tests -p "*_test.py"
python3 -m unittest discover -s core/src/tests -p "*_test.py"
python3 -m unittest discover -s .claude/skills/tui-workers/scripts -p "*_test.py"
```

## Live check (once, after the suites pass)

A real tui run through `drive.start` with the real Claude client hook and report.py:
- Cheap: Claude tier 3 (sonnet; auto mode available), effort low, `show = ""` (no iTerm2 pane), workdir under `/Users/francis/playground/agent-pm/work/TASK-166/tmp/`.
- The run's prompt has it start ≥ 4 background sub-agents finishing at spread times (e.g. each waits 20/40/60/80 s), wait for all, then report an outcome with the report command.
- Pass: the channel shows ≥ `1 + STOP_LIMIT` stops with `pending > 0` before the outcome (the count at which the old rule gives up), the driver sends no nudge and doesn't give up, the outcome arrives, the driver returns rc 0. Fewer pending stops → not a pass; rerun once with more, more spread sub-agents.
- Kill the tmux session right after; record observations in the plan doc.

## Out of scope

Writing back an outcome that arrives after the driver gave up (separate issue). `session_crons` (not asked for).
