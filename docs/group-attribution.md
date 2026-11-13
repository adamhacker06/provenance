# Group attribution — how multi-user projects flow through Provenance

Status: **descriptive**. This documents what `main` does as of 2026-11-13; it proposes nothing.
Proposals belong in a PRD. Where this file and the design record disagree, the design record wins:

- [`superpowers/specs/2026-08-19-program-decision-log.md`](superpowers/specs/2026-08-19-program-decision-log.md) — decisions D1–D16 ("do not re-litigate") and the open-items list.
- [`superpowers/specs/2026-08-19-git-collaboration-semantics.md`](superpowers/specs/2026-08-19-git-collaboration-semantics.md) — contributor semantics (§5), presentation rules (§6), worklist (§7).
- [`superpowers/specs/2026-08-24-split-replay-lanes-design.md`](superpowers/specs/2026-08-24-split-replay-lanes-design.md) — the replay lane grid.
- [`heuristics.md` § Group work](heuristics.md#group-work-which-heuristics-can-name-a-person) — which heuristics can name a person.

## 1. The model in one paragraph

A group is N students working in **one shared git repository**. Each student runs their own
recorder, which writes its own `.slog` into the repo's committed `.provenance/` directory, so after
a `git pull` every partner's logs travel together. Who recorded a session is read from exactly one
place — `session.start.identity`, a student key's countersignature over that session's ephemeral
key, anchored to an enrolment credential. Never from git author, `machine_id`, or the upload
filename. The server stores **one submission with N contributors**, runs heuristics **once over the
whole scope**, and then charges each flag to a person **only when all of its evidence sits in one
session whose identity verified**. Everything else stays scope-level: visible, at full severity,
with no name attached.

The precondition for any of this is **enrolment**. A partner who never enrolled records normally,
but their sessions cannot be tied to them, and the system will not guess.

## 2. End-to-end flow

### 2.1 Recorder (per student, per machine)

| Step                  | What happens                                                                                                                                            | Code                                                                               |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Session start         | `buildSessionIdentity` countersigns the session key with the student key. Every failure returns `skipped`; recording is never blocked.                  | [`session-identity.ts:140`](../packages/recorder/src/identity/session-identity.ts) |
| Shared directory      | Each `.slog` in `.provenance/` is classed `own` / `foreign` / `unattributed` by `student_ref`. A foreign file is never renamed, linked, or quarantined. | [`slog-ownership.ts:107`](../packages/recorder/src/startup/slog-ownership.ts)      |
| Partner's log arrives | The peer watcher hashes it and writes a `peer.observed` entry into **this** student's chain, so a later deletion is visible.                            | [`peer-watcher.ts`](../packages/recorder/src/wiring/peer-watcher.ts)               |
| Seal                  | The bundle packs every `.slog` in the directory, partners' included.                                                                                    | [`seal.ts:377`](../packages/recorder/src/commands/seal.ts)                         |

Multiple machines per student are first-class (D5): each machine has its own keypair, and the
shared `student_ref` groups them into one contributor.

### 2.2 Analysis (`analysis-core`, identical in browser and server)

`establishBundleContributors` ([`resolve-contributors.ts:487`](../packages/analysis-core/src/identity/resolve-contributors.ts))
stamps every session with one of three verdicts
([`identity/types.ts:111`](../packages/analysis-core/src/identity/types.ts)):

| Verdict        | Meaning                                                                          | Grouping                                                         |
| -------------- | -------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `attributed`   | Identity present and the chain verified.                                         | Grouped by `student_ref` into one contributor.                   |
| `unattributed` | No identity block. Ordinary and blameless.                                       | **Singleton per session** — never merged, never proven distinct. |
| `unverifiable` | Identity present but not verifiable (no root key, no anchor, or a failed chain). | Singleton per session — never merged into the person it claims.  |

Group-aware analysis turns on only when there are **two provably different attributed
contributors** (`hasTwoDifferentContributors`,
[`reconstruct-segments.ts:334`](../packages/analysis-core/src/index/reconstruct-segments.ts)). Then:

- file reconstruction refuses to linearize concurrent edits by two people (the file shows as
  branched rather than as one guessed sequence);
- a partner's recorded bytes arriving via git are classed `git_merge_in`, not as an out-of-editor
  edit ([`classify-external-changes.ts:862`](../packages/analysis-core/src/index/classify-external-changes.ts));
- cross-session ordering is a relation (`before | after | concurrent | same | unknown`), never a
  re-numbering ([`happens-before.ts:540`](../packages/analysis-core/src/order/happens-before.ts)).

### 2.3 Server ingest and storage

1. **Match** — the submitter is resolved from the filename (one `sid`) or from Gradescope's
   `submission_metadata.yml` (every co-submitter) —
   [`match-student.ts`](../packages/server/src/services/ingest/match-student.ts).
2. **Heuristics** — run once over the whole bundle.
3. **Contributor stage** — [`finalizeContributors`](../packages/server/src/services/contributors/finalize.ts), in order:
   1. `storeContributors` stamps the bundle and writes `submission_contributors`, merging the roster
      side (submitters) and the bundle side (attributed contributors) into **one row per person**
      ([`reconcileContributors`, `store-contributors.ts:119`](../packages/server/src/services/contributors/store-contributors.ts));
   2. `attributeFlags` writes `flags.contributor_key` — a person, or `''` for scope-level
      ([`attribute-flags.ts:60`](../packages/server/src/services/contributors/attribute-flags.ts));
   3. `applyContributorScores` writes each contributor's score
      ([`contributor-scores.ts:81`](../packages/server/src/services/contributors/contributor-scores.ts)).

Who gets a `submission_contributors` row
([`schema.ts:939`](../packages/server/src/db/schema.ts)):

| Person                                                      | Row?   | `kind`                                                  |
| ----------------------------------------------------------- | ------ | ------------------------------------------------------- |
| Enrolled, recorded, on the roster                           | yes    | `attributed`, `roster_entry_id` set                     |
| Enrolled, recorded, **not** on this semester's roster (D13) | yes    | `attributed`, `roster_entry_id` null — shown as unnamed |
| Named as a submitter but recorded nothing attributable      | yes    | `roster`, `session_count` 0                             |
| Not enrolled, not named as a submitter                      | **no** | —                                                       |
| Identity present but unverifiable                           | **no** | (its sessions are a finding of their own)               |

### 2.4 Scoring

- **Scope score** (`submissions.score_total` and friends) is unchanged and backs the cohort list.
- **Per-contributor score** (D14): with **≥ 2 contributor rows**, each person is charged only flags
  bearing their key, and scope-level flags are charged to nobody. With **exactly 1 row**, that person
  is charged everything, so a solo submission reads exactly as it did before groups existed.
- The **students rollup** lists a group submission under every partner who has a roster row
  ([`cohort/students.ts:225`](../packages/server/src/services/cohort/students.ts)).

## 3. What happens with 2, 3, 4 and 5 users

Assuming **every partner enrolled** and recorded in the shared repo:

|                                                   | 2                    | 3           | 4                        | 5                                            |
| ------------------------------------------------- | -------------------- | ----------- | ------------------------ | -------------------------------------------- |
| Submissions                                       | 1                    | 1           | 1                        | 1                                            |
| Contributor rows                                  | 2                    | 3           | 4                        | 5                                            |
| Cohort list label                                 | `A + B`              | `A + B + C` | four names joined by `+` | five names joined by `+`                     |
| Overview "Contributors" list                      | 2 entries            | 3           | 4                        | 5                                            |
| A paste / AI-extension / shell flag               | names whoever did it | same        | same                     | same                                         |
| A flag whose evidence spans two sessions, or none | scope-level, no name | same        | same                     | same                                         |
| Replay code lanes (split on by default)           | 2                    | 3           | **3 lanes + 1 rail**     | **3 lanes + 2 rails**                        |
| Lane colours                                      | distinct             | distinct    | distinct                 | distinct (6 hues; a 7th person would repeat) |

Notes on the replay column
([`ReplayLanes.tsx:97`](../packages/analyzer/src/views/replay/ReplayLanes.tsx),
[`lane-groups.ts:103`](../packages/analyzer/src/views/replay/lane-groups.ts)):

- Lanes go to the first three contributors in **first-appearance order**, never ranked by activity.
- Contributors 4 and 5 appear as ribbon-only rails that a grader promotes by hand. Promotion is
  component state, not URL state, so a shared link always opens on the first three.
- The cap of three is a layout and performance decision (three Monaco editors); design §11 defers
  revisiting it.

### 3.1 When not everyone enrolled

| Situation                                                                          | Result                                                                                                                                                                                                                                 |
| ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Nobody enrolled**                                                                | Every session is an `unattributed` singleton. No group-aware analysis, no split lanes; the submission reads like a solo one with several sessions. Contributors come from the roster side only.                                        |
| **Some enrolled**                                                                  | Enrolled partners are named; unenrolled partners' flags stay scope-level (pinned by `group-submission.test.ts` "charges NOBODY when the acting partner never enrolled"). Lanes appear only if at least two enrolled partners recorded. |
| **Unenrolled partner is a named submitter** (typical Gradescope group)             | They get a `roster` row, so the group has ≥ 2 rows and scope-level flags are charged to nobody.                                                                                                                                        |
| **Unenrolled partner is not a named submitter** (typical filename path: one `sid`) | They get no row. See §4 item 2.                                                                                                                                                                                                        |

`git_unrecorded_in` (D16) can fire for an honest pair whose partner was not recording; its text says
so and names enrolment as the way to tell the two readings apart.

## 4. Known limitations

From the design record (already documented there):

1. **Ingest and recompute see different heuristic input.** Ingest runs heuristics before the
   identity stamp; recompute runs them on a stamped bundle, so the same group bundle can get different
   flags. Recorded as "worth a decision of its own" (decision log, contributor-stage section).
2. **No group roster.** Partnership is only inferred from shared artifacts; partners who never share
   a repo are invisible (decision log, open item 7).
3. **Co-submitter archive divergence is silent.** When two co-submitters' archives are not
   byte-identical, one is discarded as a duplicate with no report (open item 6).
4. **Both partners unenrolled** leaves a residual recorder gap in `.slog` ownership
   ([`slog-ownership.ts`](../packages/recorder/src/startup/slog-ownership.ts) header).

Found while writing this document:

5. **The sole-contributor rule counts rows, not people — confirmed by a failing test.** If one
   partner enrolled and the other neither enrolled nor was named as a submitter, the submission has
   exactly one contributor row, so `scoreContributors` charges that enrolled partner every flag —
   including the scope-level ones from the unenrolled partner's sessions. The flag itself is
   correctly left unnamed; only the per-contributor score (and therefore the students rollup)
   absorbs it. Reproduced in `group-submission.test.ts` ("an unenrolled partner who is not a named
   submitter"): the enrolled partner scores 20.48 where their own sessions account for 15.53, the
   4.95 difference being the partner's `large_paste`. What the sole contributor _should_ be charged
   when unattributed sessions exist is a product decision, not yet made.

The rest were found by reading the code; none has a test yet: 6. **Graders cannot see per-partner attribution in the UI.** Per-contributor scores are in the API
(`SubmissionContributorSchema`) but the submission Overview shows only the scope score, and
`FlagRowSchema` carries no `contributor_key`, so the flag list cannot say whom a flag is charged
to. 7. **Lane hues lean blue/violet** (blue, violet, pink, indigo, fuchsia, sky). Not checked visually
for a five-person group. 8. **Jupyter notebooks** have no group-specific handling.
