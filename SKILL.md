---
name: fable-playbook
description: Use before starting any non-trivial coding task, such as fixing a bug, crash, error, or failing test; adding or changing a feature; deleting, cleaning up, overwriting, or migrating files or data; choosing a design; reviewing code or a plan; long multi-step work; or confirming that something is done, fixed, or working. Load it before the first edit or destructive command, even when the request already names the fix. Skip only for a trivial mechanical edit such as a typo or rename.
---

# Fable Playbook

A working method for engineering tasks: understand the goal, gather evidence, choose the smallest sufficient approach, prove the result. It is written as checkable steps that any assistant or person can follow. It teaches method and adds no capability (see [Limitations](#limitations)).

Three rules hold at every task size:

1. **Evidence before claims.** A fact is something you observed this session: a file you read, a command you ran, a statement from the user. Everything else is an assumption, and you label it.
2. **Smallest sufficient change.** Meet the acceptance criteria and stop.
3. **Written, executed, verified.** Three different states. Report the one you reached (§6).

After a context reset, compaction, or handoff, reload this file before continuing.

## 0. Start here: scale effort to the task

Pick a path first. Default to light.

**Light path**, for clear, small, reversible work in code you have read:

1. Restate the goal in one line.
2. Read the code you will change and its callers.
3. Make the change.
4. Run the check that exercises it.
5. Report what you changed and what you observed.

No written plan, option comparison, or clarifying questions.

**Deep path**, as soon as any one trigger is true:

| Trigger | Looks like |
|---|---|
| Hard to reverse | Deletes or migrates data, changes a schema or public API, deploys, sends or publishes, touches auth, secrets, or money |
| Ambiguous | Two reasonable readings of the request lead to different work |
| Wide blast radius | Shared code with many callers, or a change crossing module or service boundaries |
| Unfamiliar | Code, library, or domain you have not read or verified this session |
| Contradicted | A fix failed, a check surprised you, or evidence disagrees with the plan |
| Long | More steps than fit in one pass, or work that will outlive this context |

The deep path adds written acceptance criteria and a fact/assumption/unknown list (§1), two or more compared options (§3), verified increments with a ledger (§4), and explicit stopping criteria (§6).

Spend the depth on the part that tripped the trigger, then return to the light path. A risky migration inside a routine feature needs a careful migration, not a ceremonious feature.

## 1. Understand the task

Write four things. On the light path, the goal and one check are enough.

- **Goal**: the outcome the user wants, stated as something observable. "Orders page loads in under one second", not "add caching".
- **Constraints**: what must stay true, such as interfaces, existing behavior, performance, deadlines.
- **Non-goals**: what is out of scope. Naming them keeps the change small.
- **Acceptance criteria**: pass/fail checks someone could run. If you cannot write one, you do not understand the task yet.

Label what you know: **Fact** (observed this session), **Assumption** (believed, unchecked), **Unknown** (needed, missing). Turn the assumptions and unknowns the task depends on into facts by looking (§2). The rest go through this rule.

**Ask or assume.** Look first: a question the repository, docs, or history can answer is not a question for the user.

- **Ask** one focused question when all three hold: the answer changes what you would build, you cannot find it by looking, and a wrong guess is costly or hard to undo. Include your default so a one-word reply is enough: "Should the export include archived orders? I'll default to no, matching the list view."
- **Assume** otherwise: state the assumption in one line, take the reading that is cheapest to change later, and keep that choice in one place in the code.
- **Always ask** before a destructive or outward-facing action the user has not authorized.

**Check that the request fits the problem.** When a request names a mechanism ("add a retry", "raise the timeout", "add a cache") instead of an outcome, find out what outcome it serves and whether the evidence shows the mechanism produces it. If not, tell the user in two sentences, propose the alternative, and let them choose.

## 2. Investigate before acting

**Read before writing.** Read the code you will modify, every caller of it (search for the symbol), its tests, and one existing example of the same pattern in this codebase. Done when you can list every place the change must reach.

**Rank evidence.** When sources disagree, the higher one wins:

1. Behavior you observed: output, logs, test results, a debugger.
2. The code and configuration here, at the installed version.
3. Official documentation for that version.
4. Memory, training knowledge, blog posts, forum answers.

A claim resting only on level 4 is an assumption: check it or call it unverified.

**Check perishable facts at the source.** Signatures, command-line flags, config keys, defaults, version numbers, prices, and anything called "latest" go stale. Confirm them in the lockfile, the installed package, `--help`, or current docs.

**Separate competing explanations.** Before accepting an explanation, write one rival. For each, write a prediction the other does not make, then run the check that separates them. Evidence both predict proves neither.

> Example: an endpoint is slow. A slow query predicts slow-query log entries at any load. Waiting for a free connection predicts a clean log and latency that climbs with concurrency. One look at the log under load separates them.

**The first plausible answer is a hypothesis.** It becomes a fact when it survives a check that could have failed.

**Stop investigating** when you can state the conclusion, the observations behind it, and what would disprove it, and the next lookup would not change your decision.

## 3. Choose an approach

**Light path:** use the pattern this codebase already uses for the same job.

**Deep path:** list two or three realistic options, always including the smallest patch and what already exists (this codebase, the standard library, an installed dependency). Compare in this order unless the user sets another:

1. **Correctness**: meets every acceptance criterion, edge cases included.
2. **Failure impact**: what breaks, for whom, and how visibly when it goes wrong.
3. **Reversibility**: the cost of undoing it.
4. **Simplicity**: new concepts, moving parts, dependencies.
5. **Maintainability**: matches existing patterns, so the next reader can follow it.
6. **Cost**: time to build, cost to run.

Decision rules:

- Take the smallest option that meets every criterion. Extra complexity needs a requirement that exists today.
- On a tie, take the more reversible option.
- **One-way door** (data loss, public contract, migration, release): get explicit confirmation, prepare a rollback, rehearse on a copy.
- **Two-way door**: decide and move. Trying costs less than analyzing.
- Add an abstraction, option, or dependency for a present need only. Generalize at the third similar case.

Spend more before deciding in three cases:

- **Prototype** when the choice hinges on an empirical unknown (does the library support X, is it fast enough) that a short throwaway experiment answers and discussion cannot. Fix the question and a time limit first.
- **Investigate further** when being wrong costs more than looking and an open unknown could flip the choice.
- **Ask a person** when the deciding fact lives outside the repository (priorities, budget, compliance, production behavior) or the trade-off is a preference.

Record the decision in two lines: the option, the criterion that decided it, and the observation that would make you revisit it.

## 4. Execute reliably

**Work in verified increments.** Each step is small enough to check alone and leaves the system working. A step you cannot check is too large or lacks a check: split it or add one. On the light path the whole task is one increment.

**Order by risk.** Do the most uncertain piece first, since it is the likeliest to overturn the plan. Get a thin end-to-end slice working before polishing any part.

**Keep a ledger on long tasks**, in a file when the work may outlive this context:

- goal and acceptance criteria
- done, in progress, next, and what each next step depends on
- assumptions, each marked checked or unchecked
- open questions
- decisions and their reasons

Update it when the state changes. After a reset or handoff, re-read it and confirm the real state (`git status`, the diff, a test run) before continuing.

**Reassess when evidence contradicts the plan.** Triggers: a check fails unexpectedly, an observation breaks an assumption, a step costs several times what you expected, or you are patching the same spot a third time. Stop, write down what is now known, and return to §2 or §3.

**Stay in scope.** Change only what the task requires. Keep existing behavior, interfaces, formatting, and comments. Report unrelated problems and fix them only when asked. Before finishing, read the full diff: every hunk traces to an acceptance criterion.

**Look before destroying.** Before a delete, overwrite, reset, force, or migration: list exactly what it affects (dry run, `ls`, `SELECT` before `DELETE`), search code, config, and scripts for what still uses it and keep anything referenced, confirm you are authorized, and confirm a way back exists. If there is no way back (untracked files, no backup), make one first by moving or copying the targets aside, or stop and ask.

## 5. Debug systematically

1. **State the gap.** Expected behavior and its source (spec, test, user). Observed behavior with exact error text, input, environment, version. Keep what the reporter observed; treat what they concluded as a hypothesis.
2. **Reproduce.** Get a repeatable trigger, ideally one command or a failing test. Intermittent failure: measure the failure rate over many runs first, so "fixed" means a rate of zero, not one lucky pass. Performance problem: the baseline measurement is the reproduction. No reproduction: gather logs, inputs, and environment, add instrumentation, and report any fix as unverified.
3. **Localize.** Find where correct becomes incorrect. Trace the bad value backward from the symptom, or bisect by commit (`git bisect`), by input (halve it), or by code (disable half). Check recent changes first.
4. **Hypothesize.** List two or more candidate causes, each with a prediction the others do not make (§2).
5. **Test.** Run the cheapest separating check first. Change one thing at a time and record each result.
6. **Confirm.** The cause explains every symptom, including why the code worked before or works elsewhere, and its prediction could have failed.
7. **Fix the root cause.** Ask "why did that happen?" until you reach a change that makes the bad state impossible, not merely invisible. These are masks until shown otherwise: swallowing the error, a retry, sleep, or longer timeout, a null guard at the crash site, special-casing the failing input, loosening or deleting the test. A mask is acceptable only as a labeled temporary mitigation with the root cause reported as open.
8. **Add a regression check.** A test that fails before the fix and passes after. Watch it fail first: a test that was never red proves nothing. Then run the wider suite.
9. **Verify the real execution path.** Confirm the fixed code is what ran: rebuilt, restarted, deployed, right branch, right config, no stale cache.

**Three failed fixes means stop editing.** Something about the system is not what you think. Return to step 1, re-check each assumed fact, and consider that the reproduction or the design is wrong. Still stuck: ask for help and include what you ruled out.

## 6. Review and verify

**Use three words precisely.**

| State | Meaning | Report as |
|---|---|---|
| Written | The change exists. Nothing ran. | "Implemented, not run." |
| Executed | It ran: built, tests ran, command completed. | "Ran X, result Y." |
| Verified | An observation shows the acceptance criterion is met on the real path. | "Verified [criterion] by [observation]." |

"Should work" means written.

**Check against the original request.** Re-read the user's own words, because requirements drift during long work. Map each acceptance criterion to an observation made after your last edit.

**Name your proxies.** Passing tests show the feature works only if a test covers the path. A passing mock says nothing about the real integration. A clean build or typecheck says nothing about behavior. No error is not correct output. When a proxy is the best you have, say which one.

**Probe where changes break.** Check the items your change touches:

- Edge cases: empty, one, many, maximum; missing or null; duplicates; non-ASCII text; time zones and date boundaries; concurrency; a failing external call; missing permissions.
- Boundaries: callers, API contracts, serialization, database schema, config and environment.
- Regressions: existing tests, lint, typecheck, build, neighboring features.

**Reviewing a change, yours or someone else's.** Read the stated intent, the whole diff, and the callers of what changed. Judge correctness first, then failure handling, scope, and maintainability. Give each finding a location and a concrete failing scenario (this input produces this wrong result), marked confirmed (you ran or traced it) or suspected. A finding with no scenario is a preference: drop it unless it breaks a written project convention.

**Report honestly.** Lead with the outcome and its state. Then what changed, what was verified and how (command and result), what was not verified and why, assumptions made, known limitations, and unrelated problems noticed. Show the output of any failure.

**Stopping criteria.**

- **Done**: every acceptance criterion has a verifying observation or is listed as unverified with the reason; no known regression; the diff is in scope; open unknowns are stated.
- **Not done**: a criterion is only written or executed, a failed check is unexplained, or the thing has never run.
- **Stop investigating**: the next check would not change the decision or the report; or the remaining risk is small, stated, and reversible; or only the user can unblock you, so report and ask.
- **Change approach**: the last two or three checks gave no new information.

## 7. Failure modes

Each row has a warning sign to notice, a check to run, and a recovery if it already happened.

| Failure | Warning sign | Check | Recovery |
|---|---|---|---|
| **Premature certainty** | "The problem is X" said before any check that could disprove X. One explanation considered. "Obviously", "must be". | Name the observation that would prove you wrong and look for it. Write one rival explanation. | Restate the claim as a hypothesis, run the separating check, correct what was built on it. |
| **Invented facts or APIs** | A function, flag, config key, path, or version came from memory and you cannot point to where you saw it. | Find the exact symbol in the repo, installed package, `--help`, or current docs. Run or typecheck the call. | Replace it with the verified form, tell the user which claim was wrong, re-check what depended on it. |
| **Unnecessary complexity** | A new abstraction, option, or dependency with one user. The reason is "flexibility" or "later". A diff far larger than the request. | For each new piece, name the acceptance criterion that needs it today. | Delete back to the smallest sufficient version. Offer the removed idea as a follow-up. |
| **Symptom-only fix** | The edit sits where the error surfaced, not where the bad state began. It adds a catch, retry, sleep, guard, or special case. You cannot say why the bug happened. | State the causal chain from root cause to symptom in one sentence. | Remove the mask or label it a temporary mitigation, keep tracing upstream (§5), report the root cause as open. |
| **Irrelevant changes** | The diff has reformatting, renames, "while I'm here" refactors, version bumps, or deleted comments. | Read the complete diff and map each hunk to an acceptance criterion. | Revert the unmapped hunks and list them as suggestions. |
| **Success without evidence** | "Done", "fixed", or "tests pass" with no output from this session, or from a run before your last edit. | For each claim, point to the command and the output seen after the final edit. | Retract the claim, run the check now, report the true state (§6). |
| **Weakened check** | You are editing a test, assertion, lint rule, or type to make a failure go away. | Show the old expectation was wrong on its own merits. If you cannot, the code is wrong. | Restore the check, fix the code, tell the user what was changed and restored. |

Three related failures are handled where they occur: testing a stale artifact (§5 step 9), drifting from the request (§6), and thrashing on repeated fixes (§5).

## Worked examples

Read [references/worked-examples.md](references/worked-examples.md) when your task resembles one of these and you want the method applied end to end:

- an ambiguous feature request (§1, §3)
- a bug whose symptom points at the wrong cause (§2, §5)
- an architecture choice with more than one reasonable option (§3)

For a second opinion on work in progress, the user can run `/advisor-fable` where it is installed: a read-only review against this playbook by whichever model is currently selected.

## Limitations

- **Method, not capability.** This improves process: fewer unverified claims, smaller diffs, better questions. It adds no knowledge, reasoning depth, context size, or tool access to the model running it, and does not turn one model into another. It is general engineering practice, not a model's private reasoning or hidden instructions.
- **Bounded by the environment.** Where code cannot run, results stay at "written". Say so.
- **Project rules come first.** User instructions, repository conventions, and files such as `CLAUDE.md` or `AGENTS.md` override anything here.
- **Thresholds are defaults.** "Two options" and "three attempts" are starting points. Adjust them with a stated reason.
- **General, not specialist.** Security, performance, and framework-specific work need a domain guide as well.
- **A checklist is not proof.** Ticked boxes show the process was followed, not that the result is correct.

## Checklist

**Before acting**

- [ ] The goal is an observable outcome, and I can name the check that proves it. ([§1](#1-understand-the-task))
- [ ] I picked a path, and no deep trigger is being ignored. ([§0](#0-start-here-scale-effort-to-the-task))
- [ ] I read the code I will change and its callers, and confirmed perishable facts at the source. ([§2](#2-investigate-before-acting))
- [ ] My assumptions are written down, and nothing irreversible is unconfirmed. ([§1](#1-understand-the-task), [§4](#4-execute-reliably))
- [ ] Deep path only: two options compared, smallest sufficient one chosen. ([§3](#3-choose-an-approach))

**Before declaring done**

- [ ] I re-read the original request, and each criterion maps to an observation. ([§6](#6-review-and-verify))
- [ ] The real path ran after my last edit. ([§5](#5-debug-systematically), [§6](#6-review-and-verify))
- [ ] I read the full diff, and every hunk belongs to the task. ([§4](#4-execute-reliably))
- [ ] Bug fix only: a test went red, then green, and the wider suite ran. ([§5](#5-debug-systematically))
- [ ] My report gives each claim as written, executed, or verified, and lists what is unverified. ([§6](#6-review-and-verify))
- [ ] No warning sign from the failure table applies. ([§7](#7-failure-modes))
