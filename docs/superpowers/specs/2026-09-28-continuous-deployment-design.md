# Continuous deployment into SAP development systems via abapGit — design

Status: draft v3 for review. Date: 2026-09-28.

Items marked **[pending]** are recommended defaults awaiting the maintainer's confirmation.

## 1. Goal

Give SAP development systems the experience of continuous deployment, limited to what real SAP
landscapes allow. A background job **inside** the SAP system notices that a new version of an
abapGit-managed repository has been tagged in git and deploys it with an abapGit pull. The
deployment is verified, logged, and never reported as successful unless it demonstrably happened.

### Constraints that drive the design

- **Outbound-only.** Nothing may connect *in* to the SAP system. The only network path assumed is
  SAP → git host over HTTPS, possibly through a proxy. Systems with no outbound access at all are
  out of scope for v1.
- **abapGit is the engine.** The tool orchestrates abapGit; it does not re-implement git or object
  deserialisation. abapGit's classes are not a published API, so the whole adapter layer depends
  on internals (section 8).
- **Z-namespace tool.** All objects of *this tool* live in the customer namespace. The repositories
  it deploys may contain any objects abapGit can deserialise; that is abapGit's concern.
- **Public and private repositories.** "No credential" is a valid configuration.
- **Development systems.** Promotion to later systems uses the normal transport route.
- **Older releases stay in scope [pending].** The tool must work on a release with an older abapGit
  as long as the APIs in section 8 exist there.

### Non-goals (v1)

Automatic rollback, a UI, webhooks or any inbound endpoint, alerting (mail, chat), deployment to
non-development systems, deserialisation without abapGit, deployment history, the standalone
single-program abapGit variant, deleting objects that were removed between two tags (a pull does
not delete them; this is documented).

## 2. Behaviour

A batch job runs the report periodically (suggested: every 15 minutes). For each active configured
repository, **inside one lock** (section 6.3):

1. Acquire the lock. Not obtained → outcome `SKIPPED`, continue with the next repository.
2. Read the state row and resolve the repository by key.
3. Ask the credential provider to make the token available (private repositories only).
4. List the eligible tags (`TAG_SOURCE`) and choose the target (2.1).
5. Decide (2.2). `NOTHING_TO_DO`, `BASELINED` and `SKIPPED` end here.
6. `dry-run`: log outcome `DRY_RUN` with the decision and stop.
7. Obtain the **baseline snapshot** (5.1).
8. Switch the repository to the tag and deploy (`DEPLOYER`, 5.2).
9. Verify (`VERIFIER`, section 5).
10. Write state (6.2), write the log, `COMMIT WORK`.
11. Release the lock.

A failure in one repository never stops the others. At the end the report sets its exit status
(6.4).

The trigger is a **git tag**. Branch HEAD is deliberately not the trigger: a tag is a deliberate
promotion step and a clean rollback target.

### 2.1 Selecting the target tag

- `TAG_PATTERN` is an ABAP `CP` pattern (`*` any string, `+` any character, `#` escape; **matching
  is case-insensitive**, as in ABAP), matched against the tag name without `refs/tags/`. Example:
  `v*`. An empty pattern is invalid.
- A matching tag must parse with the grammar `^[^0-9]*(\d+)\.(\d+)\.(\d+)$` (any non-digit prefix,
  then exactly three numeric components, and nothing after them). Anything else — pre-releases such
  as `v1.2.0-rc1`, build metadata, four components, components too large for an integer — is
  **skipped and reported** in the run summary; it is never an error for the repository.
- Ordering is numeric per component (`v1.10.0` is newer than `v1.9.0`). If two tags have the same
  version (`v1.2.0` and `release-1.2.0`), the one whose name sorts last alphabetically wins.
- Annotated and lightweight tags are both accepted. The tag source returns, per tag, the **commit**
  the tag points to (never the annotated tag-object hash) and filters out peeled `^{}` entries.
- `TAG_SOURCE` returns two lists: eligible tags and skipped tags with a reason.

### 2.2 Deciding whether to deploy

Let `D` = `DEPLOYED_TAG` and `B` = `BASELINE_TAG` from the state row, `T` = the newest eligible tag.

| Situation | Outcome |
|---|---|
| No state row, or row without `BASELINED` set | Write a row with `BASELINED` set. If a tag exists, `BASELINE_TAG` := `T`; if none exists, leave it empty. Outcome `BASELINED`, **nothing deployed**. With `DEPLOY_ON_FIRST_RUN` set (default off) the run continues as if `D` and `B` were both empty. **[pending]** |
| `T` does not exist | `NOTHING_TO_DO` (logged). |
| `D` set and `T` newer than `D` | Deploy. |
| `D` empty, `B` set and `T` newer than `B` | Deploy. |
| `D` empty, `B` empty | Deploy (a repository baselined before its first release deploys its first release). |
| `T` equal to `D` | `NOTHING_TO_DO`. If the commit `T` now points to differs from `DEPLOYED_COMMIT`, log a warning "tag re-pointed"; **do not redeploy** — moving a released tag is a human decision. |
| `T` older than `D` | `NOTHING_TO_DO`, logged at information level. Going backwards is manual. |

Rationale for baselining: the repository may already have been pulled manually from a branch, and
silently moving it to a tag is surprising. A consequence: the first release *after* the baseline is
the first thing deployed. This is intended.

## 3. Components

Each unit has one purpose and is used through an interface, so the orchestrator can be tested with
fakes.

| Unit | Responsibility | Depends on |
|---|---|---|
| `ZIF_CDEPLOY_CONFIG` | Return the active repository configurations | config table |
| `ZIF_CDEPLOY_CREDENTIALS` | Make a repository's token available to abapGit for this process | secure store |
| `ZIF_CDEPLOY_TAG_SOURCE` | Return eligible and skipped tags, resolved to commits | abapGit git transport |
| `ZIF_CDEPLOY_STATE` | Read/write deployed-tag state, failure counters, snapshots | state table |
| `ZIF_CDEPLOY_DEPLOYER` | Switch to a tag and pull | abapGit repository API |
| `ZIF_CDEPLOY_VERIFIER` | Take a snapshot; judge the result of a pull | abapGit repository object |
| `ZIF_CDEPLOY_LOG` | Write to the application log and return a summary | application log |
| `ZCL_CDEPLOY_RUN` | Orchestrate section 2 | all interfaces above |
| `Z_CONTINUOUS_DEPLOYMENT` (report) | Batch entry point; parameter `dry-run` | `ZCL_CDEPLOY_RUN` |
| `Z_CONTINUOUS_DEPLOYMENT_SETUP` (report) | Store/rotate/delete a credential | credentials class |

Production implementations: `ZCL_CDEPLOY_CONFIG_DB`, `ZCL_CDEPLOY_CREDENTIALS_SECSTORE`,
`ZCL_CDEPLOY_TAGS_ABAPGIT`, `ZCL_CDEPLOY_STATE_DB`, `ZCL_CDEPLOY_DEPLOYER_ABAPGIT`,
`ZCL_CDEPLOY_VERIFIER`, `ZCL_CDEPLOY_LOG_BAL`. Exception class: `ZCX_CDEPLOY_ERROR` (carries a
failure class from 6.1 and a message).

### 3.1 Interface contracts (informative; exact signatures belong in the plan)

| Method | Input | Output / raises |
|---|---|---|
| `CONFIG->get_active` | – | table of config rows |
| `CREDENTIALS->provide` | repo key, credential id, repository URL | – / `ZCX_CDEPLOY_ERROR` if missing or unreadable |
| `TAG_SOURCE->get_tags` | repo key, tag pattern | `{eligible[{name,version,commit}], skipped[{name,reason}]}` / `ZCX_CDEPLOY_ERROR` |
| `STATE->get`, `->save` | repo key / state row | state row |
| `VERIFIER->snapshot` | repo key | `{deserialized_at, inactive_objects}` |
| `DEPLOYER->deploy` | repo key, tag, commit | `{pull_status, object_errors, decisions_missing}` / `ZCX_CDEPLOY_ERROR` |
| `VERIFIER->verify` | repo key, target tag+commit, snapshot, **deploy result** | `{ok, outcome, failed_check, objects}` |
| `LOG->add`, `->summary` | repo key, run outcome, text | – / printable text |

The verifier receives the deploy result because checks 3 (object errors) needs it.

**Verifier outcome:** `DEPLOYED`, `NO_CHANGE`, `FAILED`. **Run outcome** (what the log and the
report show): `DEPLOYED`, `NO_CHANGE`, `FAILED`, `SKIPPED`, `BASELINED`, `NOTHING_TO_DO`,
`DRY_RUN`. **`failed_check`:** `TIMESTAMP`, `REF`, `OBJECT_ERRORS`, `INACTIVE`, `LOCAL_CHANGES`,
`NEEDS_DECISION`, or initial.

### 3.2 Naming

Package `Z_CONTINUOUS_DEPLOYMENT`; tables `Z_CDEPLOY_REPO`, `Z_CDEPLOY_STATE`; lock object
`EZ_CDEPLOY`; application log object `ZCDEPLOY`, subobject `RUN`. The prefix `CDEPLOY` keeps names
inside DDIC limits (tables 16, classes and interfaces 30, lock objects 16 characters).

## 4. Configuration and state

### 4.1 `Z_CDEPLOY_REPO` — one row per watched repository

| Field | Type | Meaning |
|---|---|---|
| `REPO_KEY` | abapGit repository key (12 characters) | Identity. Mirrors abapGit's own key type. |
| `TAG_PATTERN` | string | See 2.1. |
| `CRED_ID` | 40 characters | Reference to a stored credential. Empty = public repository. |
| `DEPLOY_ON_FIRST_RUN` | flag | See 2.2. Default off. |
| `ACTIVE` | flag | Per-repository switch. |
| `DESCRIPTION` | 80 characters | Free text. |

- The repository must already be registered in abapGit; URL, package and branch live in abapGit.
- Repositories are addressed by **key, never by a name substring**.
- Keys are assigned per system by abapGit, so rows are created per system. The table is delivery
  class `A` with a **generated maintenance view (SM30)** and is not transported by default.
- **Transport request [pending]:** the tool does not keep its own request. It uses the repository's
  own abapGit setting `transport_request`, maintained in abapGit's UI. If the setting is empty and
  the package is transportable, the run fails with `NEEDS_DECISION` and a clear message. Rotating a
  released request is an abapGit-settings change, documented in the README. The batch user must be
  allowed to add objects to that request.

### 4.2 `Z_CDEPLOY_STATE` — one row per repository

| Field | Meaning |
|---|---|
| `REPO_KEY` | Key. Also the lock argument. |
| `BASELINED` | Set once the first-run rule has been applied. |
| `BASELINE_TAG` | Newest tag at baselining time; empty if there was none. |
| `DEPLOYED_TAG` | Last tag whose deployment was **verified**. Empty until then. |
| `DEPLOYED_COMMIT` | Its commit. |
| `DEPLOYED_AT` | Timestamp of the verified deployment. |
| `FAIL_TAG` | Tag of the current failing attempt; empty if none. |
| `FAIL_COUNT` | Consecutive failures for `FAIL_TAG`. |
| `LAST_ERROR` | Last failure text (never contains a credential). |
| `LAST_ERROR_AT` | Timestamp of it. |
| `SNAP_DESERIALIZED_AT` | `deserialized_at` before the **first** failed attempt for `FAIL_TAG`. |
| `SNAP_INACTIVE` | Inactive objects before that attempt (string, one name per line). |

State holds the current picture only, not a history.

### 4.3 Credentials for private repositories

- abapGit normally asks for credentials in a dialog and keeps them in session memory. A batch job
  has no dialog, so `CREDENTIALS->provide` reads the credential and hands it to abapGit's login
  manager for the repository URL **before** any git request — both tag listing and pull use it.
- A credential is a **user name and a token**, stored together as one secure-store item under the
  key `CRED_ID`. Neither is written to a table, the log, the state table, or a message.
- The secure store's access list allows reading by **one program: the credentials class**. Both
  `TAG_SOURCE` and `DEPLOYER` obtain credentials through `CREDENTIALS->provide`, not directly.
- Recommended token: fine-grained, read-only, limited to the single repository.
- `Z_CONTINUOUS_DEPLOYMENT_SETUP` stores, rotates and deletes credentials through that class. It
  never displays one.
- abapGit may still raise a dialog in batch if a credential is rejected. Batch runs without a
  dialog terminate that step; it is classified as an authentication failure (6.1).

### 4.4 Network prerequisites (documented, not implemented)

Proxy settings are inherited from abapGit. The git host's certificate chain must be trusted in the
system's certificate store. The batch user needs authorisation for abapGit deserialisation and for
the repository's transport request.

## 5. Verification

### 5.1 Baseline snapshot

`VERIFIER->snapshot` records the repository's `deserialized_at` and its currently inactive objects.
The rule that protects against false success:

- **Fresh attempt** (`FAIL_TAG` is empty or differs from the target tag): take a new snapshot and,
  before deploying, store it in `SNAP_*` only if the attempt then fails.
- **Retry of a failed attempt** (`FAIL_TAG` equals the target tag): **do not re-snapshot.** Use the
  stored `SNAP_*` from the first failed attempt, so leftovers from that attempt are not mistaken
  for pre-existing state.
- On success, or when a different tag is attempted, `FAIL_TAG`, `FAIL_COUNT` and `SNAP_*` are
  cleared.

### 5.2 Unattended pull policy

The pull is `deserialize_checks( )` then `deserialize( checks, log )`. In dialog a person answers
what `deserialize_checks` reports; in batch nobody can. The policy **[pending]**:

- **Overwrite decisions:** never auto-overwrite an object that has local changes. If
  `deserialize_checks` reports objects to overwrite, do not pull; fail with `LOCAL_CHANGES` and
  list the objects. Rationale: this is a development system and other developers' uncommitted
  work is there.
- **Any other required decision** (package, requirements, transport, warnings that need an
  answer): do not pull; fail with `NEEDS_DECISION` and name what is missing.
- **Nothing to deserialise:** if abapGit's repository status shows no difference between the
  remote files at the target tag and the local objects, no pull is issued and the result is
  `NO_CHANGE` (subject to checks 2–4).

### 5.3 The checks

The tag is recorded as deployed only if the verifier says `DEPLOYED` or `NO_CHANGE`, and all of the
checks that apply hold. `NO_CHANGE` still requires checks 2, 3 and 4.

1. **Timestamp moved** (applies only when a pull was issued): `deserialized_at` differs from the
   snapshot. If not, `FAILED` (`TIMESTAMP`) — this is the silent no-op.
2. **Requested ref is checked out:** the repository's selected ref equals `refs/tags/<tag>` and its
   remote commit equals the commit the tag was resolved to in step 4. Otherwise `FAILED` (`REF`).
   A tag that moves between steps 4 and 8 also fails here and is retried next run.
3. **No object-level errors:** the pull log contains no error for any object and `deploy` did not
   raise. Otherwise `FAILED` (`OBJECT_ERRORS`), listing the objects.
4. **No new inactive objects:** no object is inactive after the pull that was not inactive in the
   snapshot. Objects inactive before the snapshot (other developers' work) are ignored. Otherwise
   `FAILED` (`INACTIVE`).

`NO_CHANGE` is **forbidden** when `FAIL_TAG` equals the target: after a failed attempt, an empty
status may mean the failed pull already wrote everything, so the outcome is `DEPLOYED` only if
checks 2–4 hold against the stored snapshot, and otherwise `FAILED`.

### Side effect on the repository's selected ref

After a pull the repository stays selected on the tag, so a later manual pull from the abapGit UI
pulls the tag, not the branch. This is intended and documented; the tool never restores the branch.

## 6. Failure handling

### 6.1 Failure classes

| Failure | Behaviour |
|---|---|
| Remote unreachable, TLS error | `FAILED`; no tag known, so `FAIL_TAG` unchanged; `LAST_ERROR` updated. |
| Authentication rejected, or `CRED_ID` set but no secure-store entry | `FAILED`; message names `CRED_ID`; as above. |
| `REPO_KEY` unknown, repository offline or without URL | `FAILED`; as above. |
| `TAG_PATTERN` empty | `FAILED`; as above. Inactive rows are skipped silently. |
| No tag matches, or none parse | `NOTHING_TO_DO` (skipped tags reported). |
| `LOCAL_CHANGES` or `NEEDS_DECISION` (5.2) | `FAILED` with the failed check; `FAIL_TAG` set; the ref may already be on the tag. |
| Switch succeeds, pull raises | `FAILED`; `FAIL_TAG` set; the next run repeats (5.1). |
| Pull "succeeds", verifier fails | `FAILED` with the failed check; as above. |
| Lock not obtained | `SKIPPED`; nothing written. |
| Locked objects, unusable transport request, missing authorisation | `FAILED`; cause logged; `FAIL_TAG` set. |
| Job cancelled or terminated | Locks are released when the session ends; unprocessed repositories are handled next run. |

### 6.2 What each outcome writes to the state row

| Outcome | Written | Not touched |
|---|---|---|
| `BASELINED` | `BASELINED`, `BASELINE_TAG` | everything else |
| `NOTHING_TO_DO`, `SKIPPED`, `DRY_RUN` | nothing | everything |
| `DEPLOYED`, `NO_CHANGE` | `DEPLOYED_TAG/COMMIT/AT`; clear `FAIL_*`, `SNAP_*`, `LAST_ERROR*` | – |
| `FAILED` with a target tag | `FAIL_TAG`, `FAIL_COUNT`+1 (reset to 1 for a new tag), `LAST_ERROR*`; `SNAP_*` only on the first failure for that tag | `DEPLOYED_*`, `BASELINE_*` |
| `FAILED` without a target tag | `LAST_ERROR*` only | `DEPLOYED_*`, `FAIL_*`, `SNAP_*` |

"Deployed fields unchanged" is what "state unchanged" means throughout: a failed attempt never
changes `DEPLOYED_*`. There is no alerting in v1; consumers read the application log, the job's
spool and `FAIL_COUNT`.

### 6.3 Locking and rollback

The lock object `EZ_CDEPLOY` locks `Z_CDEPLOY_STATE` by `REPO_KEY`. It is requested **before**
step 2 with `_SCOPE = 1` and `_WAIT = space` (a busy lock returns immediately), and released only
in step 11. Scope 1 is required: with the default scope 2 the lock would be released at the first
`COMMIT WORK`, which abapGit's deserialisation performs. The lock's purpose is to stop two
overlapping runs of this tool (for example a job longer than its period). It does **not**
serialise against a person pulling the same repository in abapGit's UI at the same time. That
case may succeed or fail; the verifier cannot tell whose pull moved the timestamp, and this is a
documented limitation.

No automatic rollback. If a tag fails part-way the system may be partly updated; recovery is the
next tag, or re-running once the cause is fixed.

### 6.4 Reporting and exit status

One application-log entry per repository per run: run outcome, tag, failed check if any. The
report prints the same summary to the spool first, then saves the application log (`BAL_DB_SAVE`),
writes the state rows and issues `COMMIT WORK`, and only then decides the exit status.

The report ends with message type `E` if **any** repository ended `FAILED`, so the job appears as
cancelled in the job overview. Runs with only `DEPLOYED`, `NO_CHANGE`, `NOTHING_TO_DO`,
`BASELINED`, `SKIPPED` end normally. `dry-run` never sets the error status and never calls the
deployer, but does contact the remote, read credentials and write log entries marked `DRY-RUN`.

## 7. Testing

**Unit tests** (ABAP Unit; no network, no abapGit; fakes for every interface):

- decision table 2.2, every row, including first run with and without `DEPLOY_ON_FIRST_RUN`,
  baselining with no tags then a first release, tag re-pointed, older tag
- tag grammar: matching but unparseable tag skipped; pre-release skipped; `v1.10.0` beats
  `v1.9.0`; equal versions tie-break; case-insensitive pattern
- each verifier check fails on its own and yields the right `failed_check`
- `NO_CHANGE` requires checks 2–4; `NO_CHANGE` is rejected when `FAIL_TAG` equals the target
- **retry after partial failure:** first attempt fails, second attempt uses the stored snapshot and
  cannot pass on leftovers; a different newer tag resets `FAIL_*` and `SNAP_*`
- state writes match the table in 6.2 for every outcome
- unattended policy: overwrite required → `LOCAL_CHANGES`, no pull issued; other decision →
  `NEEDS_DECISION`
- lock not obtained → `SKIPPED`; `ACTIVE` off → skipped
- one repository throws → the others still run; exit status is error iff any failed
- `dry-run` → deployer not called, no state written, entries marked
- **a credential never appears in any log message, state field or exception text**, asserted over
  the output of every fake

**Integration tests** (development system, throwaway repository):

- pull a tag into a local package and into a transportable package using the repository's
  transport request setting
- pull a tag that is already deployed → `NO_CHANGE`; a deployer fake that reports success without
  pulling → verifier rejects it
- private repository with a token, public without; token rejected → authentication failure
- two registered repositories whose names overlap: only the configured key is touched
- re-run after a failed pull with the repository already on the target ref → pulls again and is
  judged against the stored snapshot
- two overlapping runs: the second is `SKIPPED` and the lock is still held for the whole pull

## 8. Source-checked facts about abapGit (read in source, not run live)

Checked in source on two ABAP releases, both with the abapGit developer edition installed
(source dated April 2025 and June 2024). The interfaces read were identical on both apart from one
newer local-settings field the tool does not use. All of this is abapGit **internals**, not a
published API, and can change between versions:

- Repository keys are `c length 12` (`ZIF_ABAPGIT_PERSISTENCE`).
- The repository object exposes `ms_data-deserialized_at` and the selected ref.
- `select_branch` stores any ref name; the push code treats `refs/tags/*` specially.
- `get_git_transport( )->branches( url )->get_tags_only( )` exists (commented "for potential future
  use").
- A pull is `deserialize_checks( )` then `deserialize( is_checks, ii_log )`.
- `ZCL_ABAPGIT_LOGIN_MANAGER=>set( iv_uri, iv_username, iv_password )` stores credentials in a
  class-level table (per internal session).
- The secure store has `SECSTORE_INSERT_ITEM`, `SECSTORE_READ_ITEM`, `SECSTORE_DELETE_ITEM` with an
  access list restricting readers by program. Their release status for customer use is unchecked.
- abapGit's local settings contain a per-repository `transport_request`.

## 9. Still to verify before the plan (a spike, outcomes recorded here)

1. An actual pull of a tag through `select_branch` + `deserialize` in a batch context, including
   what the login manager needs for a private repository, and that `refs/tags/*` behaves as a tag
   at runtime.
2. How `get_tags_only` represents annotated tags (tag object vs. peeled commit).
3. Which API answers "does the remote at this ref differ from the local objects" (abapGit's
   repository status), and what it returns for an up-to-date repository. Check 1 and `NO_CHANGE`
   depend on this.
4. Whether `deserialized_at` moves on a pull that deserialised something and whether it can fail
   to move on a legitimate one.
5. Which errors abapGit raises as exceptions and which it puts into the log; check 3 must cover
   both.
6. What `deserialize_checks` reports for unattended runs, and how to detect "decisions missing".
7. Whether changing the selected ref to a tag is honoured when the repository object was built
   earlier in the same process.
8. Release status of the `SECSTORE_*` function modules for customer use.

## 10. Repository layout

The repository is itself the abapGit repository for package `Z_CONTINUOUS_DEPLOYMENT`. ABAP source
lives under `src/` in abapGit format, folder logic `PREFIX`, generated by SAP's own serialisation
and never hand-written. abaplint runs in CI. Changes land through pull requests.
