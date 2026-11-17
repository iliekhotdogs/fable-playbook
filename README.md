# fable-playbook

A Claude Code skill that gives the model a working method for coding tasks: understand the goal, gather evidence, make the smallest sufficient change, and prove the result before claiming it is done.

It adds no capability. It changes process: fewer unverified claims, fewer bugs hidden behind guards, and a look before anything destructive.

## What it does

- **Scales effort to the task.** Small, clear, reversible work takes a light path: read, change, check, report. Work that is hard to reverse, ambiguous, wide-reaching, unfamiliar, contradicted by evidence, or long takes a deep path with acceptance criteria, compared options, and a progress ledger.
- **Investigates before acting.** It reads the code to change and its callers, ranks evidence (observed > code > docs > memory), and checks perishable facts such as flags and versions at the source.
- **Debugs to the root cause.** Reproduce, localize, test competing hypotheses, fix the cause. A catch, retry, null guard, or special case counts as a mask, not a fix. After three failed fixes it stops and re-checks its assumptions.
- **Looks before destroying.** Before a delete, overwrite, or migration it lists what is affected, searches for anything that still uses it, and makes sure a way back exists.
- **Reports honestly.** Every claim is labelled written, executed, or verified, and anything unverified is listed.

## When it loads

Before any non-trivial coding task: a bug, crash, or failing test; a feature; deleting, cleaning up, or migrating data; a design choice; a code or plan review; long multi-step work; or confirming that something works. It skips trivial mechanical edits such as typos and renames.

## Install

Copy the folder into your personal skills directory, or into a project's `.claude/skills/`:

```bash
git clone https://github.com/iliekhotdogs/fable-playbook ~/.claude/skills/fable-playbook
```

Claude Code picks it up automatically. To force it for one task, start the prompt with `/fable-playbook`.

## Evaluation

The skill was behaviour-tested on Claude Haiku 5.5 with headless runs across five small fixture repositories. Each task ran in three arms: no skill, skill installed (loads itself), and skill forced with `/fable-playbook`.

| Result | Finding |
|---|---|
| Triggering | Loaded itself on 41 of 42 risky tasks and 0 of 10 typo fixes. |
| Masked bug | A crash request that invites a null guard over a real parsing bug. Bug hidden: 5 of 14 runs without the skill, 1 of 13 self-loaded, 0 of 9 forced. Root cause fixed: 4 of 14, 6 of 13, 8 of 9. |
| Destructive cleanup | "Delete the old exports": 0 of 20 runs deleted permanently without asking or backing up first. |
| Overhead | A typo fix took about 12 s instead of 7 s. |

Limits: small samples (9 to 14 runs per arm), Haiku only, and other plugins and hooks disabled during testing. The skill did not fix arithmetic errors: wrong dollar totals appeared in 8 of 36 masked-bug runs with or without it. The ledger, architecture, and three-failed-fixes sections are untested.

### Running the eval

Requires the `claude` CLI and Python 3. Each run uses Claude Code's sandbox and a scrubbed environment.

```bash
cd eval
python3 make_fixtures.py
python3 harness.py run --out runs/v1 --reps 3
python3 harness.py grade --out runs/v1
```

`run` accepts `--arms control,natural,forced`, `--scenarios typo,mask,reuse,verify,destroy`, `--reps N`, `--jobs N`, and `--skill <folder>` to test another version. Fixtures and runs go to `$FABLE_EVAL_DIR` (default `/tmp/fable-playbook-eval`), which must not be inside a `.claude/` folder because Claude Code denies edits there. `grade` prints verdicts and writes each run's final message to `finals.md`. The verdicts come from pattern matching, so read `finals.md` before trusting one.

A full run (5 scenarios, 3 arms, 3 reps) is about 45 Haiku runs. Some runs leave backup copies in the sandbox temp directory; remove them afterwards.
