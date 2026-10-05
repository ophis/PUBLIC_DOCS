# TASK-166 plan

RESUME: phase=S9 worktree=/Users/francis/playground/agent-pm/work/TASK-166/src/agent-pm branch=TASK-166-tui-runner-stop-hook-background-tasks-2 base_ref=34da64917e7b5a6bd966b636f0fac67bf015c415 review_round=0 spec_file=/Users/francis/playground/agent-pm/work/TASK-166/src/agent-pm/docs/.autopilot/TASK-166-spec.md

Spec and plan live in gitignored `docs/` (not committed).

## Progress

- S1: reused the existing worktree (not on main); base_ref 34da649.
- decision(field name): `report.py stop --pending KEY`, the claude hook passes `background_tasks`; over hardcoding it in report.py - requirement keeps Claude field names out of core outside the client; dissent: none
- decision(stdin): report.py reads stdin only with `--pending` - a hand-run `stop` never blocks on a terminal; dissent: none
- decision(docs check): hooks.md says `background_tasks` lists in-flight tasks, present (maybe `[]`) when the registry is reachable, omitted otherwise.
- S3 panel: core=[architecture,spec-fitness] +optional=[] (security skipped: no authz/secrets/IO trust change) transport=Workflow
- S3 r0: architecture=PASS spec-fitness=PASS -> converged; non-blockers folded into spec (give-up helper, clock start, hook-input shape note in core/CLAUDE.md, Event.pending on stop only, live-check pass rule)
- S5: Tasks 1-3 complete (9f6ba28, 183d5df, a5ea8a0, 6308caa); per-task reviews clean; deferred minors in .superpowers/sdd/TASK-166-plan/progress.md
- S6: suites orchestrator 478 OK, core 313 OK, tui-workers 61 OK; `background_tasks` in core/src non-test only at clients/claude.py:45
- S6 live check (tmp/live_check.py: drive.plan dummy-tester/echo, tier 3, show "", custom prompt: 5 background haiku sub-agents sleeping 25-125 s): run 1 channel stops pending 5,4,3,2,1 then outcome; driver stayed, no nudge; outcome rejected only because the test prompt omitted --deliverable (echo is local). Run 2 (prompt adds --deliverable): stops pending 5,4,3,2,1, outcome, result rc=0 status=done, 0 nudges in the transcript. Both tmux sessions killed.
- S7 panel: core=[correctness,requirement-fidelity,doc] +optional=[code-quality,test] (performance skipped: one monotonic() call per poll) transport=Workflow
- S7 r0: correctness=PASS requirement-fidelity=PASS doc=PASS code-quality=PASS test=PASS -> converged
- S8 skipped (run instruction: keep the commits)
- S9 residual non-blockers: report.py stop --pending crashes on a closed stdin (manual calls only); WAIT_LIMIT message says 'after the last progress report' when none came; time.monotonic pauses during macOS sleep, so the 2 h is awake time; Tui.done gives up as a side effect; `type(pending) is int`; README:79 dense; README:77 'done once its outcome arrives'

## Implementation plan

Spec: `/Users/francis/playground/agent-pm/work/TASK-166/src/agent-pm/docs/.autopilot/TASK-166-spec.md`

### Global Constraints

- Work only in `/Users/francis/playground/agent-pm/work/TASK-166/src/agent-pm` on branch `TASK-166-tui-runner-stop-hook-background-tasks-2`, absolute paths / `git -C <worktree>`; before any write assert `git -C <worktree> branch --show-current` is that branch. No other clone, worktree or branch; never touch `main`, never force-push.
- Follow `CLAUDE.md` and `core/CLAUDE.md`; match the surrounding code (dense docstrings, few comments, stdlib only).
- Tests first: write the failing test, see it fail, then the code. **No mutation testing**: never substitute a known-wrong value into code under test, by edit or runtime reassignment. Fake time by patching `time.monotonic` (stdlib); fake stdin by patching `sys.stdin`.
- `background_tasks` appears in `core/src/` only in `clients/claude.py` and tests.
- Constants: `WAIT_LIMIT = 2 * 60 * 60` (seconds) in `core/src/drive.py`; `STOP_LIMIT` stays 3.
- Verify (all must pass), from the worktree:
  `python3 -m unittest discover -s orchestrator/src/tests -p "*_test.py"`;
  `python3 -m unittest discover -s core/src/tests -p "*_test.py"`;
  `python3 -m unittest discover -s .claude/skills/tui-workers/scripts -p "*_test.py"`.
- Commit messages: `TASK-166: <what changed>`, ending with `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. After each commit run exactly `git -C /Users/francis/playground/agent-pm/work/TASK-166/src/agent-pm push -u origin TASK-166-tui-runner-stop-hook-background-tasks-2`.
- Temp files: `/Users/francis/playground/agent-pm/work/TASK-166/tmp/`.

### Task 1: Stop lines carry pending background work

Files: `core/src/report.py`, `core/src/clients/base.py`, `core/src/clients/claude.py`, `core/src/drive.py` (`report_event` only), `core/src/tests/drive_test.py`.

Produces:
- `report.py --to CH stop [--pending KEY]` → `{"kind": "stop", "pending": N}` / `{"kind": "stop"}` per spec §1; stdin read only with `--pending`.
- `clients.Event.pending: int = 0`, docstring: meaningful on `stop` only.
- `drive.report_event`: `pending` an `int`, not `bool`, ≥ 0 → `Event("stop", pending=N)`; else `Event("stop")`.
- Claude Stop hook command: `report_command(CORE_SCRIPTS, params) + " stop --pending background_tasks"`.
- `report.py` module docstring shows `stop [--pending KEY]`.

Tests first (in `drive_test.py`, next to the existing report/stop tests):
- `stop --pending background_tasks` with stdin `{"background_tasks": [{}, {}]}` → line `{"kind": "stop", "pending": 2}`, exit 0, stdout `report.py: stop reported`; `{"background_tasks": []}` → `pending: 0`.
- With `--pending`: stdin `""`, `"not json"`, `"[1]"`, `"{}"`, `{"background_tasks": 3}`, undecodable bytes → `{"kind": "stop"}`, exit 0.
- Without `--pending`: `sys.stdin` patched with an object whose `read` raises → still `{"kind": "stop"}` (stdin untouched).
- `report_event('{"kind": "stop", "pending": 2}') == Event("stop", pending=2)`; `pending` `true`, `-1`, `"2"`, `1.5` → `Event("stop")`; existing `{"kind": "stop"}` → `Event("stop")`.
- Update `test_interactive_adds_the_stop_hook_after_the_config_flags`: command ends `stop --pending background_tasks`.

Commit: `TASK-166: stop lines carry the hook's pending background work`

### Task 2: Tui waits through pending stops; 2 h quiet limit

Files: `core/src/drive.py` (`WAIT_LIMIT`, `Tui`, module docstring, `STOP_LIMIT` comment), `core/src/tests/drive_test.py` (`TuiRunner`, `FakeTui` if needed).

Consumes: `Event.pending` (Task 1).

Produces:
- `WAIT_LIMIT = 2 * 60 * 60` with a one-line comment.
- `Tui`: `begin` records the clock after `tui.start` succeeds; `seen`: progress → stops = 0 and clock restarts; stop with `pending > 0` → ignored (no nudge, no count); other stops → today's rule. `done(outcome_arrived)`: `outcome_arrived` → True; else if not given up and `time.monotonic() - since > WAIT_LIMIT` → give up. One helper sets `gave_up` and prints `drive.py: <reason>; session <name> left open: tmux attach -t '=<name>'` for both paths (STOP_LIMIT reason text unchanged: `no outcome after <n> stops`; quiet reason e.g. `no outcome 2 h after the last progress report`).
- Docstrings: module (tui done once the outcome arrives or it gives up), `Tui` (pending stops, quiet limit).

Tests first (`TuiRunner`):
- `1 + STOP_LIMIT` and more stops with `pending > 0`, then an outcome → done, rc 0, no `send`.
- Pending stops interleaved with pending-0 stops: nudge on the first pending-0 stop only; give-up after `STOP_LIMIT` more pending-0 stops regardless of pending ones in between.
- Quiet limit: fake `time.monotonic` (stdlib patch; e.g. a clock a step advances) past `WAIT_LIMIT` with no progress → Result(0, None, "the run returned no outcome"), attach line on stderr, session not killed (`calls` has no `kill`), `main` exits 1.
- Progress restarts the clock: progress at t = WAIT_LIMIT - 10, then t = WAIT_LIMIT + 10 → no give-up; give-up only after WAIT_LIMIT since that progress.
- Not exceeded at exactly `WAIT_LIMIT` (`>`).
- An outcome arriving in the same poll the limit passes → done with the outcome (rc 0, status done).
- Existing STOP_LIMIT tests stay green with the unchanged message.

Commit: `TASK-166: the tui runner waits through pending stops and gives up after 2 h quiet`

### Task 3: Docs

Files: `README.md` (line 79 paragraph), `core/CLAUDE.md` (Add a client › 3), any other doc stating the old rule (grep `STOP_LIMIT`, `wait timeout`, `nudge`, `gives up`, `turn end`).

Consumes: Tasks 1–2's behavior and names.

Produces:
- README:79: the hook reports each turn end with the run's pending background work; a stop with pending work gets no nudge and isn't counted; otherwise the nudge + `STOP_LIMIT` rule; `drive.py` also gives up after 2 h without a progress report (`WAIT_LIMIT`, from the run's start if none), the same way; drop "There is no wait timeout"; keep "a dialog nobody answers waits in the pane" only if still true (it waits until the quiet limit).
- core/CLAUDE.md step 3: `interactive` appends `stop` at each turn end, with `--pending KEY` when the hook's stdin JSON has its in-flight background work as a list at top-level `KEY` (claude: `background_tasks`); another input shape needs another option; tui neither nudges nor counts a stop with pending work; replace "it has no wait timeout" with the quiet limit.
- Brief: no restatement of code.

Tests: none (docs); the three suites still pass.

Commit: `TASK-166: docs for pending stops and the 2 h quiet limit`
