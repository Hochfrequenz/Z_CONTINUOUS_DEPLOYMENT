# Handover - the live to-do list

Read this first. Update it in the same pull request as the work it describes: section 0 is the only part
that changes often. Last updated: 2026-09-29.

## 0. Where things stand

| Item | State |
|---|---|
| Design spec | On `main`, draft v3.7: `docs/superpowers/specs/2026-09-28-continuous-deployment-design.md` |
| Plans 00 (overview), 01 (spike), 02 (core engine) | On `main`: `docs/superpowers/plans/` |
| Plans 03 (adapters), 04 (reporter and reports) | **Outlines only**, inside plan 00. Deliberately not detailed until the spike has answered its questions. |
| Implementation | **None.** No object exists on any system, `src/` does not exist yet. |
| abaplint CI (PR #3) | May still be open. It is the prerequisite of plan 02's "lint like CI" steps. Check `gh pr list`. |
| Decisions taken | Overwrite policy per repository, default `ALWAYS`; older ABAP releases in scope; GitHub (`github.com`) only. |
| Still `[pending]` in the spec (defaults are implemented, each is one method to change) | suffix ordering; first run baselines only; `MAX_ATTEMPTS` 5 with back-off; transport request from abapGit's own setting |

**The code in plan 02 has never run.** It parses and type-checks under `abaplint.json`, its language
assumptions were probed on a real 7.50 system, and its tests were traced by hand - nothing more. Expect
defects on the first run; the red/green steps exist for that.

## 1. Read in this order

1. `docs/superpowers/specs/2026-09-28-continuous-deployment-design.md` - what and why (sections 1, 2, 5, 6).
2. `docs/superpowers/plans/2026-09-28-00-overview.md` - order, gates, the ABAP workflow, **section 4:
   findings that contradict the spec**, and section 4a: language facts checked on a real system.
3. The plan you are about to execute (01 or 02).
4. `CLAUDE.md` - the repository's rules for agents (project notes at the bottom are filled in).

## 2. Next actions, in order

1. **Merge or review PR #3** (abaplint pipeline) if it is still open.
2. **Fold overview section 4 findings 1 to 3 into the spec** in a small spec PR - *before* implementing
   plan 02, or the code will contradict the spec (plan 02, Task 0, Step 0).
3. **Ask the maintainer** (see section 3) before touching a system.
4. **Run plan 01 (spike) and plan 02 (core engine)** - they are independent and can run in parallel. The
   spike decides two gates: G1 (does the lock survive a pull: the one-minute polling period depends on it)
   and G2 (does the timestamp move on an empty pull).
5. **Only then detail plans 03 and 04** against the recorded spike results.

## 3. Ask the maintainer before you start

- Which system to develop on first, and which workbench request to put objects into. (Never release a
  request without permission. Pass the request, not the task.)
- Whether to create the fixtures of spike task S0 (a public and a private GitHub fixture repository, tags,
  two fine-grained PATs stored by the maintainer, never by an agent).
- Whether running the unit tests on a system is wanted now. It was **declined once to save tokens**; only
  short probe snippets were run.
- Anything that is token-expensive: the maintainer wants to be asked first.

## 4. Working rules (they come from the maintainer; the spec's section 10 has the same)

- **Squash merge only.** An agent runs at least one round with a *separate* review agent before a pull
  request is marked ready; until then it stays a draft. Re-review your own fixes: fixes made after a review
  round introduced new defects every time so far.
- Credentials are **PATs, never passwords**. Only tagged versions are deployed. Timestamps are UTC
  `TIMESTAMPL`, never `sy-datum`/`sy-uzeit`. Respect DDIC length limits (classes and interfaces 30, tables and
  lock objects 16) - a too-long test method name is the most common failure.
- **This repository is public.** No system names, hosts, clients, customer or person names, tokens, and no
  name of a private repository in commits, PRs or documents. Say "the older system" and "the newer system".
- abapGit XML is never hand-written: create on a system through ADT, export, commit what SAP serialised.

**Do not:** implement or detail plans 03 and 04 until `docs/spike/RESULTS.md` records the two gates G1 and G2
(plan 00, sections 1 and 2). Do not run anything on an SAP system - probe classes and unit tests included -
without asking the maintainer first.

## 5. Tooling facts worth knowing

- The ADT loop is in plan 00, section 3. `mcp__sap-adt__export_package` needs the companion export package on
  the system.
- **abaplint** (`abaplint.json` exists only once the abaplint CI pull request, #3, is merged) runs only the rules
  listed in it, needs `dependencies` to know any standard class, and its `object_naming` patterns lose a
  backslash (use character classes). Details are comments in that file.
- **Short snippets on a real system are cheap and worth it** (plan 00, section 4a): one throwaway class in the
  local package implementing `IF_OO_ADT_CLASSRUN`, run with `mcp__sap-adt__run_class`, then deleted and
  confirmed gone. It found two real defects that lint could not. A classrun class cannot be a job step.
- A `sap_abapgit_pull` that says "Pull successful" proves nothing: check that `DESERIALIZED_AT` moved, and
  compare unit-test counts between the systems (plan 02, Task 9).

## 6. Unknown, not assumed

Every abapGit call in the spec's section 8 was *read in source*, never run. Section 9 and plan 01 list what
must be measured. Nothing about a 7.40 system was verified (the probes ran on 7.50; the abaplint floor covers
syntax only).
