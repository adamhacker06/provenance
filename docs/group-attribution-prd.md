# PRD — Group projects: attribution fixes and instructor visualizations

Structure follows Atlassian's PRD template
(<https://www.atlassian.com/agile/product-management/requirements>): project specifics, goals,
background, assumptions, requirements, design, questions, not doing.

Companion: [`group-attribution.md`](group-attribution.md) describes what `main` does today. This
document only proposes. Nothing here is built.

## 1. Project specifics

|                    |                                                                                                                                                   |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Issue**          | Multi-user attribution for group projects (opened by go-bears, 2026-09-26)                                                                        |
| **Assignees**      | saahilsukhija, adamhacker06                                                                                                                       |
| **Status**         | **Draft**: not reviewed, not approved                                                                                                             |
| **Target release** | Undecided. The issue expects one track (visualizations _or_ core) before Fall 2026 ends and the rest in Spring 2027; triage (issue step 4) picks. |
| **Design record**  | [Decision log](superpowers/specs/2026-08-19-program-decision-log.md) D1–D16 are settled and not reopened here.                                    |

## 2. Goals and objectives

1. **A grader can act on one partner without implicating another.** For a group of 2–5, the
   grader can see which findings belong to which partner, which belong to nobody, and why.
2. **Per-partner numbers are trustworthy.** A partner's score never includes a finding from
   another partner's sessions, whatever the enrolment mix.
3. **The same bundle gets the same findings** whether it was just ingested or recomputed.
4. **Solo submissions do not change.** Every requirement below must leave a solo, non-git
   submission's flags, score and display identical (collaboration spec §6 negative rule).

## 3. Background and strategic fit

Group support already exists (D9, D14, split-lane replay; see
[`group-attribution.md`](group-attribution.md) §2). The gaps are of two kinds:

- **Built but not shown.** Per-contributor scores and per-flag attribution are computed and
  stored, but the analyzer only shows the scope score, and the flags API does not say whom a flag
  is charged to. A grader cannot currently use the main thing D14 was built for.
- **Built but wrong at the edges.** One confirmed scoring bug (a failing `it.fails` test in
  `group-submission.test.ts`), one documented ingest-vs-recompute divergence, and two silent cases
  from the decision log's open items.

Many group projects cap at 5 people. Replay was designed around a practical cap of 3 code lanes,
so 4- and 5-person groups are where the visual work is thinnest.

## 4. Assumptions

| #   | Assumption                                                                                                | If wrong                                                                                           |
| --- | --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| A1  | Groups work in one shared git repo with every partner's recorder writing to the committed `.provenance/`. | Partners with no shared artifact stay invisible (see C4).                                          |
| A2  | In a typical course, at least some partners will not have enrolled.                                       | If everyone enrols, C1 matters less and V1/V2 matter more.                                         |
| A3  | Groups are 2–5 people; 6+ is rare enough to defer.                                                        | Palette and lane work needs revisiting.                                                            |
| A4  | Graders read the submission Overview and the replay first, and the API second.                            | V1/V2 placement changes.                                                                           |
| A5  | Analysis fixtures can build 3–5 verified contributors, as they do 2 (`build-collab-scope.ts`).            | **Unverified**: only two fixed refs exist today; may need fixture work before any 4–5 person test. |

## 5. Requirements

Importance: **Must** = the track is not useful without it; **Should** = expected; **Could** =
only if time allows. Every requirement ships with tests (CLAUDE.md) and, where it changes routes,
ingest, or the API, an `/architecture` update in the same PR.

### Track V — Instructor visualizations

| #   | User story                                                                                                                                                                | Importance | Notes / success measure                                                                                                                                                                                                                                                                                                                                                    |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| V1  | As a grader, on each flag I can see **who it is charged to** (a named partner, an unnamed partner, or "scope-level: not attributable to one person") with the reason.     | Must       | Flags already carry `session_id` and the summary carries each session's contributor verdict, so this may need no API change. How to avoid a second copy of the attribution rule is Q8. Wording must follow collaboration spec §6 Rule 2. Success: every flag in a 2–5 person fixture shows exactly one of the three labels, and the label matches `flags.contributor_key`. |
| V2  | As a grader, on a group submission's Overview I see **each partner's score and severity** beside the scope score, with a note that scope-level flags count toward no one. | Must       | Data is already in `SubmissionContributorSchema`; no API change. Success: the sum of displayed partner scores ≤ scope score, and a solo Overview is pixel-identical apart from no new section.                                                                                                                                                                             |
| V3  | As a grader replaying a 4–5 person group, I can choose **which partners get the code lanes**, and a link I share opens with the same choice.                              | Should     | Today promotion is component state and the cap is 3 (split-lanes design §11). Success: `?lanes=` (or similar) round-trips; a 5-person fixture can show any 3.                                                                                                                                                                                                              |
| V4  | As a grader, I can **tell 5 lane colours apart** at a glance.                                                                                                             | Should     | Current hues are blue, violet, pink, indigo, fuchsia, sky, and must stay clear of the orange/red/amber/emerald finding colours. Success: a contrast/distinguishability check on the 5 used hues, light and dark.                                                                                                                                                           |
| V5  | As a grader, the cohort list tells me a submission is a group and **how many partners are unenrolled**, without opening it.                                               | Could      | Possibly a facet or chip. Must not read as a finding (D13).                                                                                                                                                                                                                                                                                                                |

### Track C — Core correctness

| #   | User story                                                                                                            | Importance                 | Notes / success measure                                                                                                                                                                                                                        |
| --- | --------------------------------------------------------------------------------------------------------------------- | -------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| C1  | As a grader, an enrolled partner's score **never includes findings from an unenrolled partner's sessions**.           | Must                       | Confirmed bug: the sole-contributor rule counts rows, not people. The fix needs Q1 answered. Success: the `it.fails` test flips to `it`, and the solo `contributor-scores` tests still pass unchanged.                                         |
| C2  | As a grader, **ingest and recompute produce the same flags** for the same group bundle.                               | Must                       | Ingest runs heuristics before the identity stamp; recompute after. Needs Q2. Changes the ingest pipeline → `/architecture` update. Success: an ingest-then-recompute test on a 2-person bundle with `git_merge_in` yields identical flag sets. |
| C3  | As staff, I am told when **two co-submitters' archives of one group differ** instead of one being silently discarded. | Should                     | Decision log open item 6. Shape: a coverage fact, never a flag.                                                                                                                                                                                |
| C4  | As staff, I can **declare groups** so partners who share no repo are still linked.                                    | Could (likely Spring 2027) | Decision log open item 7: needs a course-supplied group roster, a product decision, and a schema change.                                                                                                                                       |

## 6. User interaction and design

No wireframes yet; to be added before building. Constraints any design must respect:

- Reuse the existing flag panel and drawer (`FlagDashboardPanel` / `HeuristicDetailDrawer`) for V1
  rather than adding a second flag list.
- Lane hues must never read as a finding colour (`contributor-palette.ts` header).
- "Unenrolled" and "unattributed" are blameless states and must never be styled as warnings;
  "identity did not verify" is a finding and is styled as one (`identity/types.ts`).
- `/local` has no server-side scores. Whether V1/V2 must also work there is Q6.

## 7. Questions — things we need to decide or research

| #   | Question                                                                                                                                                                                                                                              | Blocks     | Owner                |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- | -------------------- |
| Q1  | When a submission has **one contributor row but also unattributed sessions**, what is that contributor charged: only flags in their own sessions, or everything except flags from other sessions? Who decides, since this is a scoring-policy change? | C1         | TBD                  |
| Q2  | Should ingest **stamp identity before heuristics**? Doing so changes which flags group bundles get at ingest, a product change the decision log flagged as "worth a decision of its own".                                                             | C2         | TBD                  |
| Q3  | Which track ships in Fall 2026, and what is the actual end-of-term date?                                                                                                                                                                              | Scheduling | Issue author         |
| Q4  | Is the 3-lane cap a hard limit (performance) or a default?                                                                                                                                                                                            | V3         | TBD                  |
| Q5  | Where would a group roster come from: Gradescope group metadata, a course CSV, or the manifest?                                                                                                                                                       | C4         | TBD                  |
| Q6  | Must V1/V2 also work on `/local` (in-browser, no server)?                                                                                                                                                                                             | V1, V2     | TBD                  |
| Q7  | Does V5's "unenrolled count" risk being read as an integrity signal? D13 says it must not.                                                                                                                                                            | V5         | TBD                  |
| Q8  | For V1, should the analyzer get `contributor_key` from the server (an API contract change), or should the server's `contributorKeyForSession` move into `analysis-core` so both share one copy of the rule?                                           | V1         | Assignees + reviewer |

## 8. What we're not doing

- **Running heuristics per contributor.** Closed by analysis (D14); it manufactures findings
  against honest pairs (`contributor-scope-boundary.test.ts`).
- **Using git author, `machine_id`, or the upload filename as identity.** Ruled out (collaboration
  spec §5.1, `events.ts` "No author identity. Ever.").
- **Code-content or ML classification** of who wrote what (CLAUDE.md; recorder PRD NG5).
- **Retroactively merging** rows from the old per-co-submitter fan-out.
- **A Jupyter recorder.** Not built (architecture roadmap: "JupyterLab recorder · not built"); a
  notebook's unit is a cell, which forces an event-model decision before any code. Stretch goal
  only: at most a scoping note this term.
- **Changing solo behaviour** in any way.
- **Real-time or multi-grader collaboration** features (analyzer v3 PRD non-goal).
