# Triage — group attribution: Fall 2026 vs Spring 2027

Draft, local, not approved. Item IDs (V1–V5, C1–C4, Q1–Q7) refer to
[`group-attribution-prd.md`](group-attribution-prd.md). Written 2026-11-13.

## 1. Inputs

- **Calendar.** Today is Fri 2026-11-13. The end-of-term date is **not confirmed** (PRD Q3); this
  triage assumes roughly four working weeks, with Thanksgiving week (from 2026-11-26) lost.
- **People.** Two assignees, part-time.
- **"Done" is expensive here.** Per CLAUDE.md: tests, typecheck, lint, a reviewable diff, and an
  `/architecture` update for any route, ingest, or API change, all in the same PR.
- **The critical path is decisions, not code.** C1 and C2 are small once decided, but Q1 and Q2
  are scoring and pipeline policy that the assignees should not decide alone.

## 2. Findings that change the PRD

1. **V1 may not need an API change.** `FlagRowSchema` already carries `session_id`
   (`api-schemas.ts`, `FlagRowSchema`), and `SubmissionSummarySchema` already carries the per-session
   contributor stamp (`by_session`). The analyzer could label each flag from data it already
   has, on both the server-backed view and `/local`. The catch: that would be a **second copy** of
   the server's `contributorKeyForSession` predicate (`attribute-flags.ts:60`), the "two
   implementations waiting to disagree" pattern `lane-mode.ts` warns about. Options: (a) add
   `contributor_key` to `FlagRowSchema` (contract change, server only); (b) move the predicate into
   `analysis-core` so server and analyzer share one copy. This is a new question, **Q8**.
2. **V2 must not ship before C1.** Per-partner scores are exactly the numbers C1 shows to be wrong
   when an unenrolled partner exists. Showing them first puts a known-wrong number in front of
   graders. (The bug already reaches graders today through the students rollup, which sums
   per-contributor scores.)
3. **Fixing C1 or C2 does not fix stored rows.** Per-contributor scores and flags are rewritten
   only by a recompute. Any fix needs a stated plan for existing submissions (run a semester
   recompute, or accept stale rows until the next one).

## 3. Sizing

Size: **S** ≈ 1–2 days, **M** ≈ 3–5 days, **L** = more than a week, including tests and review.
These are rough estimates from reading the code, not commitments.

| ID  | Item                                            | Size               | Blocked by               | Depends on                    | Risk                                                                                    | Proposed                                       |
| --- | ----------------------------------------------- | ------------------ | ------------------------ | ----------------------------- | --------------------------------------------------------------------------------------- | ---------------------------------------------- |
| C1  | Sole-contributor scoring bug                    | **S** once decided | **Q1**                   | —                             | Low: a pure function plus one `it.fails` → `it`; needs a recompute plan                 | **Fall 2026** if Q1 is answered by ~2026-11-20 |
| V1  | Show whom each flag is charged to               | **M**              | Q8 (new), Q6             | —                             | Low–medium: UI plus maybe an API field and the `/architecture` readpath                 | **Fall 2026**                                  |
| V4  | Five distinguishable lane colours               | **S**              | —                        | —                             | Low: verify visually in light and dark                                                  | Fall 2026 (filler)                             |
| V3  | Choose replay lanes; keep the choice in the URL | **S–M**            | Q4 only if the cap moves | —                             | Low if the 3-lane cap stays                                                             | Fall 2026 if time, else Spring                 |
| V2  | Per-partner scores on the Overview              | **S**              | —                        | **C1**                        | Shows wrong numbers without C1                                                          | With or after C1                               |
| C2  | Ingest/recompute give the same flags            | **M**              | **Q2**                   | —                             | Medium: changes ingest output for group bundles; ingest pipeline `/architecture` update | Spring 2027                                    |
| V5  | Group / unenrolled indicator on the cohort list | **M**              | Q7                       | —                             | Must not read as a finding (D13)                                                        | Spring 2027                                    |
| C3  | Report co-submitter archive divergence          | **M**              | —                        | —                             | Needs somewhere to persist the coverage fact                                            | Spring 2027                                    |
| C4  | Course-declared group roster                    | **L**              | Q5                       | —                             | Product decision plus a schema change                                                   | Spring 2027 or later                           |
| —   | Jupyter group projects                          | **L+**             | Event-model decision     | A recorder that doesn't exist | —                                                                                       | Scoping note at most; build later              |

## 4. Options for Fall 2026

The issue expects one track. All three options below fit about four weeks.

| Option                                     | Contents                 | Needs before starting                               | Leaves for Spring      |
| ------------------------------------------ | ------------------------ | --------------------------------------------------- | ---------------------- |
| **A. Visualization, no decisions needed**  | V1 + V4 (+ V3 if time)   | Q8, which the two of you can settle with a reviewer | C1, V2, C2, V5, C3, C4 |
| **B. Core**                                | C1 + C2                  | **Q1 and Q2**, both policy                          | All of V               |
| **C. Smallest useful slice (recommended)** | C1 + V1, then V2 if time | **Q1** (plus Q8)                                    | C2, V3, V4, V5, C3, C4 |

**Why C:** C1 is a live correctness bug in a grader-visible number (the students rollup), and its
code fix is small; the risk is only the decision. V1 is the most-wanted visualization and doesn't
wait on policy. Together they deliver the issue's goal for the common 2–5 person case: a grader
can tell which partner a finding belongs to, and the per-partner totals are right. C crosses the
two tracks; if the issue author holds to "pick one", A is the fallback.

**Fallback rule:** if Q1 has no answer by ~2026-11-20, switch to Option A, so the term isn't
spent waiting on a decision.

## 5. Spring 2027 backlog (in proposed order)

1. C2: decide Q2 first, then align ingest with recompute (with an `/architecture` ingest update).
2. V2, if it did not land in Fall.
3. V3 + V4 for 4–5 person groups, if not done.
4. V5, after Q7.
5. C3: coverage fact for divergent co-submitter archives.
6. C4: group roster, after Q5.
7. Jupyter: a scoping doc on how `doc.change` maps onto notebook cells, before any recorder work.

## 6. Asks of the issue author (go-bears)

1. The end-of-term date (Q3).
2. A decision on **Q1**, or who owns it.
3. Approval of Option C (or A/B).
4. Whether a cross-track slice is acceptable, or the issue's "pick one" is firm.
