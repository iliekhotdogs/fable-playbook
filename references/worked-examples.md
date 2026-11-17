# Worked examples

Three teaching examples for the [playbook](../SKILL.md).

**All three are illustrative.** The scenarios, evidence, and numbers are invented to show the method. None records a real session, a real tool result, or a completed test. The verification lists describe checks you would run, not checks that were run.

Each example has the same five parts: the problem and its constraints, the evidence, a common weak response and why it fails, the recommended approach with the rationale you would give the user, and the check that verifies the result.

1. [Ambiguous feature request](#1-ambiguous-feature-request-add-an-export-button): scoping, ask or assume (§1, §3)
2. [Misleading bug symptom](#2-misleading-bug-symptom-the-database-is-slow): competing explanations, root cause (§2, §5)
3. [Architecture trade-off](#3-architecture-trade-off-where-to-run-background-jobs): comparing options, reversibility (§3)

## 1. Ambiguous feature request: "add an export button"

### Problem and constraints

The whole request: "Add an export button to the orders page." The product is a web app with a REST API. Nothing is said about file format, columns, which rows, or who may export. The user wants it this week and is hard to reach.

### Evidence (hypothetical)

- Searching the repository for "export" finds an invoice export: a streaming CSV endpoint that applies the list page's current filters, and a shared `toCsvRow` helper that escapes commas, quotes, and newlines.
- The orders list has status and date filters and shows eight columns.
- The orders API also returns customer email and shipping address, which the list page does not show.
- Production row count: unknown.

### Common weak response

Either of two opposite reactions:

- **Overbuild.** A general export system with CSV, XLSX, and PDF, a column picker, and background jobs that email the file.
- **Over-ask.** Eight questions about format, columns, limits, encoding, and permissions before touching anything.

Why they fail: the first builds four unrequested features on guesses and adds dependencies, so the one wanted thing arrives late inside a large diff. The second hands the user work the repository already answers and stalls a change that is cheap to adjust later.

### Recommended approach

1. **Sort what is known.** Fact: a tested export pattern exists. Assumptions: CSV is acceptable, and "export" means the rows matching the current filters. Unknowns: whether exports may contain customer contact data, and the production row count.
2. **Apply ask-or-assume.** Format, mechanism, and filter behavior are answered by the existing invoice export and are cheap to change, so follow it. Row count stops mattering because the existing endpoint streams. Contact data is different: a file that has left the system cannot be recalled, and the repository cannot tell you the policy. That one earns a question.
3. **Ask once, with a default.** "Orders carry customer email and address that the list doesn't show. I'll export only the eight visible columns unless you want those included. OK?"
4. **Proceed on the reversible parts meanwhile.** Add an endpoint that reuses the invoice export's streaming and `toCsvRow`, with the columns defined in one list and the same permission as viewing orders.

Rationale to give the user: "I matched the invoice export so both behave the same way, and limited the file to what the page already shows. The columns live in one list, so adding more is a one-line change."

### Verification

- With the status filter set to "shipped", the file's row count equals the count shown in the list.
- The header row equals the eight visible column names, in order.
- A seeded order whose customer name contains a comma, a quote, and a newline opens as one correct cell in a spreadsheet.
- A value beginning with `=` is exported as text, not as a formula.
- A user without permission to view orders gets a 403 from the endpoint.

If the `=` check fails, the gap is in the shared helper and affects invoices too. Report it and ask before changing shared code; it is outside this task.

## 2. Misleading bug symptom: "the database is slow"

### Problem and constraints

The report: "Checkout requests have been timing out on and off since last week. The errors say the database timed out. Please add an index or raise the timeout." Checkout is revenue-critical and runs in production. There is no load-test environment.

### Evidence (hypothetical)

- Exact error text: `timeout acquiring connection from pool after 30s`.
- Median latency is normal. The 99th percentile sits at the 30-second limit.
- The database's slow-query log for the same window shows no statement over 100 ms.
- Timeouts cluster when more than about ten checkouts are in flight.
- Last week's change: tax is now calculated by calling an external tax API, and that call sits inside the order's database transaction.

### Common weak response

Raise the timeout to 60 seconds, add retries, and add an index on `orders`.

Why it fails: it acts on the reporter's conclusion without testing it, although the slow-query log already contradicts "slow query". A longer timeout makes users wait longer for the same failure. Retries add load to a resource that is already exhausted. The index speeds up statements that were never slow. The symptom would probably persist, now buried under three new variables.

### Recommended approach

1. **State the gap.** Expected: checkout completes in under two seconds at normal load. Observed: some requests fail after waiting 30 seconds for a connection. Keep the reporter's observations (timeouts, since last week) and treat "the database is slow" as a hypothesis.
2. **List rival causes with predictions that separate them.**

   | Hypothesis | Predicts |
   |---|---|
   | A. Slow query | Entries in the slow-query log. Slow even with one request at a time. |
   | B. Pool exhaustion: connections held while waiting on the tax API | Clean slow-query log. Time spent waiting for a connection grows with concurrency and with tax-API latency. |
   | C. Lock contention | Statements visibly waiting on locks, which would also show as slow statements in the log. |

3. **Run a check that could fail.** The clean slow-query log already counts against A and C. B makes a prediction nobody has checked yet: timeouts should track tax-API latency. Line up the timeout timestamps with the tax API's response times for the same window. If timeouts occur while the tax API is fast, B is wrong. Then reproduce locally with a pool of two connections, a one-second acquire timeout, the tax API stubbed to take two seconds, and five concurrent checkouts: acquire timeouts should appear, and disappear with the stub delay at zero. That reproduction becomes the regression test.
4. **Fix the root cause.** Call the tax API before opening the transaction, pass the result in, and give the external call its own short timeout. A connection is then held only for database work.
5. **Label mitigations as mitigations.** A larger pool moves the threshold without removing the cause. If it is applied to stop the bleeding, say so and keep the root cause open until step 4 ships.

Rationale to give the user: "The queries aren't slow. Requests are queueing for a database connection because each checkout holds one while it waits on the tax service. An index or a longer timeout wouldn't change that. Moving the tax call outside the transaction removes the wait."

### Verification

- The local reproduction fails before the change and passes after it. Run it before the fix and watch it fail.
- A test asserts that no external HTTP call happens while a transaction is open on the checkout path.
- Existing checkout tests pass, including the case where the tax API errors: the order fails cleanly with nothing half-written.
- The fix moves the tax calculation ahead of the commit, so add a test for a cart that changes in between: the order is rejected or the tax is recomputed.
- After deploy, connection-wait time and 99th-percentile latency at peak return to their earlier levels. Until that is observed, report the fix as executed, not as verified in production.

## 3. Architecture trade-off: where to run background jobs

### Problem and constraints

The signup endpoint sends a welcome email inline. When the mail provider is slow, signup is slow, and a failed send is lost for good. The team wants background jobs with retries. Three engineers, one Postgres database, no other infrastructure, a few hundred jobs a day.

### Evidence (hypothetical)

- Infrastructure config shows Postgres only. No Redis and no message broker are deployed.
- The user reports about 300 signups a day and thinks a tenfold increase within a year is plausible.
- The requirement that matters most: a job exists if and only if the signup committed.

### Options

| Criterion | A. Jobs table in Postgres, worker using `FOR UPDATE SKIP LOCKED` | B. Redis-backed queue library | C. Managed cloud queue |
|---|---|---|---|
| Correctness | Enqueue happens in the signup's own transaction: no lost or orphaned jobs | Enqueue is a second write: needs an outbox table or accepts rare loss | Same second-write problem as B |
| Failure impact | Queue is down only when the database is down, and then signup is down too | New failure mode: Redis unavailable | New failure mode: network or provider |
| Reversibility | Backend hidden behind one `enqueue(job)` function, so it can be swapped | Same, if kept behind the function | Same |
| Simplicity | One table, one worker loop, nothing new to operate | A new service to run, monitor, and back up | New account, credentials, permissions, local emulation |
| Throughput | Far above this load; the ceiling is an estimate, not measured here | High | Very high |
| Cost | Lowest | Hosting plus operations | Per-request fees plus setup |

### Common weak response

"Use Kafka with a separate worker service, it's the scalable choice." Or naming the most familiar tool as if it were the only option.

Why it fails: it optimizes for a load far beyond the requirement, adds infrastructure that three people must operate, and skips the criterion that decides this case, which is enqueueing inside the transaction. Presenting a single option also hides the trade-off from the person who owns the decision.

### Recommended approach

Recommend A behind a small `enqueue` function, and say when the answer would change:

"For a few hundred jobs a day on a Postgres-only stack, a jobs table is the smallest option and also the most correct one: the job is written in the same transaction as the signup, so it can't be lost or orphaned, and there is nothing new to run. I'd revisit if sustained volume reaches thousands of jobs a minute, if job rows start to bloat the database, or if you adopt Redis for another reason. The `enqueue` function keeps that switch contained. If you already ran Redis, B would be a reasonable choice too."

Ask the one question that could flip the choice: "Is there a known event that will multiply volume soon, such as a bulk import or a launch?"

If volume were the real uncertainty, a one-hour throwaway experiment measuring enqueue and dequeue rates on a copy of the schema would settle it faster than discussion.

### Verification

- **Atomicity.** A test that rolls back the signup transaction leaves no job row. A committed signup leaves exactly one.
- **Retry.** With the mail provider stubbed to fail twice and then succeed, one email is sent and the job row records three attempts.
- **No double processing.** Two workers running against 100 queued jobs send exactly 100 emails.
- **Latency.** With the mail provider stubbed to take five seconds, signup response time stays where it was.
- **Crash recovery.** A worker killed mid-job leaves the job retryable. Delivery is therefore at-least-once, so a duplicate email is possible: state that to the user and let them decide whether it is acceptable.
- The throughput ceiling is an estimate. Measure it before relying on it.
