# Continuous deployment into SAP development systems via abapGit — design

Status: draft v3.4 for review. Date: 2026-09-28.

Items marked **[pending]** are recommended defaults awaiting the maintainer's confirmation.

## 1. Goal

Give SAP development systems the experience of continuous deployment, limited to what real SAP
landscapes allow. A background job **inside** the SAP system notices that a new version of an
abapGit-managed repository has been tagged in git and deploys it with an abapGit pull. The
deployment is verified, logged, and never reported as successful unless it demonstrably happened.

### Goals

- **Only tagged versions are deployed.** Never a bare commit on `main`, never "whatever is newest".
  A tag is a deliberate act by a human; the tool only ever follows tags.
- **Fast.** From a tag being pushed to a *verified* deployment in the system: **5–10 minutes at
  most**. Anything slower loses against setting up a VPN and logging in by hand — even
  transport-of-copies usually ships within ten minutes. The budget is one polling interval plus one
  pull. This is why the tool polls every minute (section 2) and why a poll must be cheap.
- **Observable from outside.** Someone without SAP access can tell which deployments succeeded and
  which failed, and can tell that the job itself is alive (6.6). The more accessible, the better.

### Constraints that drive the design

- **Outbound-only.** Nothing may connect *in* to the SAP system. The only network path assumed is
  SAP → git host over HTTPS, possibly through a proxy. Systems with no outbound access at all are
  out of scope for v1.
- **abapGit is the engine.** The tool orchestrates abapGit; it does not re-implement git or object
  deserialisation. abapGit's classes are not a published API, so the whole adapter layer depends
  on internals (section 8).
- **Z-namespace tool.** All objects of *this tool* live in the customer namespace. The repositories
  it deploys may contain any objects abapGit can deserialise; that is abapGit's concern.
- **Public and private repositories.** "No credential" is a valid configuration. Where a credential
  is needed it is always a **personal access token (PAT)**, never a password (4.3).
- **Development systems.** Promotion to later systems uses the normal transport route.
- **Older releases stay in scope [pending].** The tool must work on a release with an older abapGit
  as long as the APIs in section 8 exist there.

### Non-goals (v1)

Automatic rollback, a UI, webhooks or any inbound endpoint, alerting (mail, chat), deployment to
non-development systems, deserialisation without abapGit, deployment history, the standalone
single-program abapGit variant, deleting objects that were removed between two tags (a pull does
not delete them; this is documented).

## 2. Behaviour

A batch job runs the report **every minute** (the shortest period a background job supports; every
5 minutes is the slowest interval that still meets the goal in section 1). A poll is one small
HTTPS request per repository — the remote's tag advertisement, not a fetch of any objects — so a
high frequency is cheap. A poll that finds nothing to do writes no log entry and prints nothing
(6.2, 6.4); it only updates a heartbeat. For each active configured repository, **inside one lock**
(section 6.3):

1. Acquire the lock. Not obtained → outcome `SKIPPED`, go to step 10 (nothing to write).
2. Read the state row and resolve the repository by key.
3. Ask the credential provider to make the token available (private repositories only).
4. List the eligible tags (`TAG_SOURCE`) and choose the target (2.1).
5. Decide (2.2) and check the attempt cap and back-off (6.1). The outcomes `NOTHING_TO_DO`
   (including "waiting for back-off"), `BASELINED` and `FAILED` (cap reached) are terminal:
   **go to step 9**. In `dry-run` mode the run logs what *would* happen (`DRY_RUN`, including
   "would baseline", "would wait for back-off" and "would stop: attempt limit reached") and goes to
   step 10, **before writing any state**. The "clean slate" test of 5.2 and `force_pull` are
   computed here, from the state row as read in step 2 — **before** the attempt marker of step 6
   changes it.
6. Take the snapshot if none is held and **write the attempt marker** (5.1), then `COMMIT WORK`.
   The lock survives the commit (6.3). Then tell the reporter the deployment started (6.6; best
   effort, after the commit).
7. Switch the repository to the tag — **always**, even if no pull is issued, because check 2 needs
   it — and deploy (`DEPLOYER`, 5.2).
8. Verify (`VERIFIER`, section 5).
9. Write the outcome state (6.2) and the log, then `COMMIT WORK`. **Every** path that ends a
   repository, terminal or not, passes through this step, so a dump in a later repository cannot
   lose the recorded outcome of an earlier one. After the commit, and **only in a run that
   called `started` successfully in step 6**, report the verdict to the reporter (6.6; best effort —
   a reporting failure never changes the outcome). Quiet polls, `BASELINED`, the attempt cap and
   failures before step 6 make no reporter call at all: no GitHub request per repository per
   minute, and no timeout to wait for when the API is unreachable.
10. Release the lock. The release also runs in a cleanup handler; the enqueue is released by the
    system when the session ends.

A failure in one repository never stops the others, with one exception: a short dump inside
abapGit cannot be caught and ends the whole job. The attempt marker (step 6) is what makes the next
run safe after such a crash. A repository that dumps on every attempt therefore delays the
repositories after it until the attempt cap (6.1) takes effect; repositories whose state has
`FAIL_TAG` set are processed **last** in each run to limit this. At the end the report sets its exit
status (6.4).

The trigger is a **git tag**. Branch HEAD is deliberately not the trigger: a tag is a deliberate
promotion step and a clean rollback target.

### 2.1 Selecting the target tag

- **One fixed rule; there is deliberately no per-repository pattern.** An eligible tag matches
  `^v(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)\.(0|[1-9][0-9]*)(-.+)?$`, matched **case-sensitively**
  against the tag name without `refs/tags/`: a literal lowercase `v`, exactly three numeric
  components **without leading zeros** (as semantic versioning requires), then optionally a hyphen
  and at least one further character. Semantic versioning is mandatory at the start of the tag;
  freedom exists only after the patch number. Developers do not get to invent tag schemes.
- Everything else is **skipped**: no leading `v`, two or four components, `release-1.2.0`,
  `my-customer-final17-2026`, `v01.2.3`, `v1.2.3-` (a hyphen with nothing after it), a suffix
  without a hyphen (`v1.2.0rc1`), and any component that does not fit an ABAP `i`
  (above 2147483647). Skipped tags are never an error for the repository; they are listed in the
  status report and logged only when the set of skipped tags changes (6.4).
- **Equality is equality of the parsed version including the suffix**; with the no-leading-zeros
  rule that is the same as equality of the tag name, so "T equals D" is well defined.
- **Ordering follows semantic-versioning precedence.** Numeric per component (`v1.10.0` is newer
  than `v1.9.0`). For equal `X.Y.Z`, a tag **without** a suffix is newer than one with a suffix
  (`v1.2.0` is newer than `v1.2.0-rc1`). Two suffixes compare identifier by identifier, where
  identifiers are separated by dots: numeric identifiers compare as numbers (`rc.10` is newer than
  `rc.2`), a numeric identifier is older than an alphanumeric one, alphanumeric identifiers compare
  as ordinal strings, and with equal leading identifiers the shorter list is older. A suffix
  without dots, like `-rc10`, is one alphanumeric identifier and orders as a string (`-rc10` is
  older than `-rc2`); use `-rc.10` if numeric ordering matters. **[pending]**
- **Any suffixed tag is a deployable release**, and a suffixed tag with a higher `X.Y.Z` displaces a
  lower plain one: a stray `v9.0.0-test` becomes `D`, after which every real release below 9.0.0
  is "older" (row 7). Recovery is in 6.5. A stored `D` or `B` that no longer parses (someone edited
  the state table) ends the repository as `FAILED` with a clear message.
- Annotated and lightweight tags are both accepted. The tag source returns, per tag, the **commit**
  the tag points to (never the annotated tag-object hash) and filters out peeled `^{}` entries.
- `TAG_SOURCE` returns two lists: eligible tags and skipped tags with a reason.

### 2.2 Deciding whether to deploy

Let `D` = `DEPLOYED_TAG` and `B` = `BASELINE_TAG` from the state row, `T` = the newest eligible tag.

**Rows are evaluated first-match, top to bottom.**

| # | Situation | Outcome |
|---|---|---|
| 1 | `FAIL_TAG` equals `T` | **Retry** (deploy). A failing tag is always retried, up to `MAX_ATTEMPTS` (6.1). |
| 2 | No state row, or row without `BASELINED` set, and `DEPLOY_ON_FIRST_RUN` is off | Write a row with `BASELINED` set and `BASELINE_TAG` := `T` (empty if no tag exists). Outcome `BASELINED`, **nothing deployed**. |
| 3 | Same as 2, but `DEPLOY_ON_FIRST_RUN` is on | Write a row with `BASELINED` set and `BASELINE_TAG` **left empty**, then continue with row 4 or 8. **[pending]** |
| 4 | `T` does not exist | `NOTHING_TO_DO` (quiet: not logged). |
| 5 | `D` set and `T` newer than `D` | Deploy. |
| 6 | `D` set and `T` equal to `D` | `NOTHING_TO_DO`. If the commit `T` now points to differs from `DEPLOYED_COMMIT`, there is a note "tag re-pointed"; **do not redeploy** — moving a released tag is a human decision. |
| 7 | `D` set and `T` older than `D` | `NOTHING_TO_DO` (quiet). Going backwards is manual. |
| 8 | `D` empty, `B` empty | Deploy (covers a repository baselined before its first release, and the first-run override). |
| 9 | `D` empty, `B` set, `T` newer than `B` | Deploy. |
| 10 | `D` empty, `B` set, `T` equal to or older than `B` | `NOTHING_TO_DO`. This is the steady state of every baselined repository until its next release. |

Rows 2 and 3 write the baseline fields together with the fields of whichever outcome follows
them, committed no later than the attempt marker (step 6) or step 9, whichever comes first.

Rationale for baselining: the repository may already have been pulled manually from a branch, and
silently moving it to a tag is surprising. A consequence: the first release *after* the baseline is
the first thing deployed. This is intended.

**Withdrawn tag after a failed attempt.** If `FAIL_TAG` is set and differs from the tag the rows
above would act on (for example the failing tag was deleted, so `T` equals `D`), there is a note
"the system may be partly at `FAIL_TAG`". The row's normal outcome still applies.

**Notes are logged only when they change.** The "tag re-pointed" note, the withdrawn-tag note and
the skipped-tag list (2.1) would otherwise repeat every minute. The run computes the text of all
current notes, hashes it, and writes a log entry only when the hash differs from `LAST_NOTE_HASH`
in the state row (4.2). The status report (6.6) always shows the current notes.

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
| `ZIF_CDEPLOY_REPORTER` | Publish deployment start and verdict to the outside (6.6); best effort | git host API |
| `ZCL_CDEPLOY_RUN` | Orchestrate section 2 | all interfaces above |
| `Z_CONTINUOUS_DEPLOYMENT` (report) | Batch entry point; parameter `dry-run` | `ZCL_CDEPLOY_RUN` |
| `Z_CONTINUOUS_DEPLOYMENT_SETUP` (report) | Store/rotate/delete a credential; operator functions `RESET_ATTEMPTS`, `CLEAR_BASELINE` and `RESET_DEPLOYED` (6.5) | credentials class, state |
| `Z_CONTINUOUS_DEPLOYMENT_STATUS` (report) | Read-only status list, one line per repository (6.6) | state |

Production implementations: `ZCL_CDEPLOY_CONFIG_DB`, `ZCL_CDEPLOY_CREDS_SECSTORE`,
`ZCL_CDEPLOY_TAGS_ABAPGIT`, `ZCL_CDEPLOY_STATE_DB`, `ZCL_CDEPLOY_DEPLOYER_ABAPGIT`,
`ZCL_CDEPLOY_VERIFIER`, `ZCL_CDEPLOY_LOG_BAL`, `ZCL_CDEPLOY_REPORTER_GITHUB`, and
`ZCL_CDEPLOY_REPORTER_NULL` (used when reporting is off). Exception class: `ZCX_CDEPLOY_ERROR` (carries a
failure class from 6.1 and a message).

### 3.1 Interface contracts (informative; exact signatures belong in the plan)

| Method | Input | Output / raises |
|---|---|---|
| `CONFIG->get_active` | – | table of config rows |
| `CREDENTIALS->provide` | repo key, credential id, repository URL | – / `ZCX_CDEPLOY_ERROR` if missing or unreadable |
| `CREDENTIALS->get_report_token` | report credential id | the token, **for the reporter only**; never handed to abapGit's login manager / raises if missing |
| `REPORTER->started` / `->finished` | repo key, tag, commit / run outcome, failed check | – (never raises) |
| `TAG_SOURCE->get_tags` | repo key | `{eligible[{name,version,commit}], skipped[{name,reason}]}` / `ZCX_CDEPLOY_ERROR` |
| `STATE->get`, `->save` | repo key / state row | state row |
| `VERIFIER->snapshot` | repo key | `{deserialized_at, inactive_objects}` |
| `DEPLOYER->deploy` | repo key, tag, commit, **`force_pull`** | `{pull_issued, pull_status, object_errors, decisions_missing}` / `ZCX_CDEPLOY_ERROR` |
| `VERIFIER->verify` | repo key, target tag+commit, snapshot, **deploy result** | `{ok, outcome, failed_check, objects}` |
| `LOG->add`, `->summary` | repo key, run outcome, text | – / printable text |

The verifier receives the deploy result because check 3 (object errors) needs it. The orchestrator
sets `force_pull` whenever an attempt has already happened since the last verified deployment (5.2).
The deployer always switches the ref; it issues a pull unless the status is empty **and**
`force_pull` is not set, and reports which it did in `pull_issued`. **The verifier is the only unit
that produces `NO_CHANGE`:** it maps "no pull issued + checks 2–4 hold" to `NO_CHANGE`.

**Verifier outcome:** `DEPLOYED`, `NO_CHANGE`, `FAILED`. **Run outcome** (what the log and the
report show): `DEPLOYED`, `NO_CHANGE`, `FAILED`, `SKIPPED`, `BASELINED`, `NOTHING_TO_DO`,
`DRY_RUN`. **`failed_check`:** `TIMESTAMP`, `REF`, `OBJECT_ERRORS`, `INACTIVE`, `LOCAL_CHANGES`,
`NEEDS_DECISION`, or initial.

### 3.2 Naming

Package `Z_CONTINUOUS_DEPLOYMENT`; tables `Z_CDEPLOY_REPO`, `Z_CDEPLOY_STATE`; lock object
`EZ_CDEPLOY`; application log object `ZCDEPLOY`, subobject `RUN`. The prefix `CDEPLOY` keeps names
inside DDIC limits (tables 16, classes and interfaces 30, lock objects 16 characters). **Every name
in this document was length-checked mechanically** (longest class: `ZCL_CDEPLOY_DEPLOYER_ABAPGIT`, 28;
longest program: `Z_CONTINUOUS_DEPLOYMENT_STATUS`, 30 of 40); the check is repeated in CI (10).

## 4. Configuration and state

### 4.1 `Z_CDEPLOY_REPO` — one row per watched repository

| Field | Type | Meaning |
|---|---|---|
| `REPO_KEY` | abapGit repository key (12 characters) | Identity. Mirrors abapGit's own key type. |
| `CRED_ID` | 40 characters | Reference to a stored PAT used to read the repository. Empty = public repository. |
| `ENVIRONMENT` | 40 characters | Name under which this system appears in outward reporting (6.6). **Must be a neutral alias** ("dev-1", "team-a-dev"), never a system ID, host, client or customer name: on a public repository it is world-readable. Empty = no outward reporting. |
| `REPORT_CRED_ID` | 40 characters | Reference to a separate PAT allowed to write deployment status (6.6). Empty = no outward reporting. Setting only one of `ENVIRONMENT` / `REPORT_CRED_ID` counts as "reporting off" and produces a note. |
| `REPORT_ON_PUBLIC` | flag | Allow outward reporting to a **public** repository. Default off (6.6). |
| `DEPLOY_ON_FIRST_RUN` | flag | See 2.2. Default off. |
| `MAX_ATTEMPTS` | integer | Attempts per failing tag before the tool stops pulling it (6.1). Default 5; a value of 0 is treated as the default. **[pending]** |
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
| `FAIL_TAG` | Tag of the current attempt or failed attempt; empty if none. Set **before** the pull (5.1), so it also marks a crashed attempt. |
| `FAIL_COUNT` | Attempts made for `FAIL_TAG`. |
| `LAST_ERROR` | Last failure text, or `attempt in progress` while a pull runs. Never contains a credential. |
| `LAST_ERROR_CLASS` | `ATTEMPT` (a deployment attempt, tied to `FAIL_TAG`), `REMOTE`, `AUTH`, `CONFIG`, or empty. Drives alert de-duplication (6.1a). |
| `LAST_ERROR_AT` | UTC timestamp of it. |
| `LAST_ALERT_AT` | UTC timestamp of the last time a failure was logged and set the error exit status (6.1a). |
| `LAST_OUTCOME` | Run outcome of the most recent run that did something other than a quiet poll. For the status report. |
| `LATEST_TAG` | Newest eligible tag seen at the last successful poll. For the status report. |
| `LAST_NOTE_HASH` | Hash of the text of the current notes (skipped tags, re-pointed tag, withdrawn failed tag); a log entry is written only when it changes (2.2). |
| `LAST_POLL_AT` | Heartbeat: UTC timestamp of the last time the job processed this repository (whatever the outcome). |
| `LAST_POLL_OK_AT` | Heartbeat: UTC timestamp of the last poll in which **tag listing succeeded**. An authentication failure is not success (it shows as `LAST_ERROR_CLASS` = `AUTH`). If this goes stale while `LAST_POLL_AT` is fresh, git is unreachable or rejecting us. |
| `SNAP_TAKEN` | Flag: the inactive-object baseline has been taken for the current run of attempts. Set with the attempt marker; cleared only on `DEPLOYED` / `NO_CHANGE`. It, **not** the emptiness of `SNAP_INACTIVE`, decides whether a baseline exists — a clean package legitimately has zero inactive objects. |
| `SNAP_INACTIVE` | Inactive objects at the start of the **first** attempt since the last verified deployment (string, one `TYPE NAME` per line; all users; restricted to the repository's package and its sub-packages). May be empty when `SNAP_TAKEN` is set. |

State holds the current picture only, not a history. There is deliberately no stored timestamp
snapshot: the timestamp for check 1 is always read fresh (5.1).

**Timestamps.** Every timestamp this tool stores, compares, logs, prints or sends is an explicit
**UTC long timestamp** (`TIMESTAMPL`, taken with `GET TIME STAMP FIELD`) and is shown with the
zone named. `sy-datum`, `sy-uzeit`, `sy-datlo` and `sy-timlo` are not used anywhere, because a bare
date or time without a zone is ambiguous across systems and daylight-saving changes. This matches
the type of abapGit's own `deserialized_at`, which is compared in check 1. The static check (10)
only forbids the bare names `DATUM`, `UZEIT`, `DATLO` and `TIMLO`; that a value really is a
`TIMESTAMPL` is a code-review item, not something abaplint can verify.

### 4.3 Credentials for private repositories

- abapGit normally asks for credentials in a dialog and keeps them in session memory. A batch job
  has no dialog, so `CREDENTIALS->provide` reads the credential and hands it to abapGit's login
  manager for the repository URL **before** any git request — both tag listing and pull use it.
- A credential is always a **personal access token (PAT)**. The tool cannot tell a PAT from a
  password, so "never a password" is enforced by design and labelling, as a policy: the setup
  report has a single secret input labelled "PAT" and no user-name or password input. It is stored
  as one secure-store item under the key `CRED_ID`, and never written to a table, the log, the
  state table, or a message. HTTP basic authentication needs a user-name part; that is a fixed
  label (`token`) which nobody enters, because GitHub ignores it when the secret is a PAT (spike
  item 9.10). **This is verified only for GitHub**; other git hosts (some need a specific user
  name) are out of scope for v1.
- The secure store's access list allows reading by **one program: the credentials class**. Both
  `TAG_SOURCE` and `DEPLOYER` obtain credentials through `CREDENTIALS->provide`, not directly.
- Recommended PAT: fine-grained, **read-only**, limited to the single repository. The PAT used for
  outward reporting (`REPORT_CRED_ID`, 6.6) is a *different* one so the pull token stays read-only.
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

`VERIFIER->snapshot` returns two independent things, treated differently:

- **`deserialized_at`** is read **fresh, immediately before every pull**, including retries. It is
  never stored. Check 1 compares against it, so a retry cannot pass on a timestamp that an earlier
  attempt already moved.
- **The inactive-object baseline** must not absorb leftovers of failed attempts, so it is kept
  across attempts:
  - If `SNAP_TAKEN` is not set (no attempt since the last verified deployment), take the baseline
    now and store it together with `SNAP_TAKEN`.
  - If `SNAP_TAKEN` is set, **keep the stored baseline**, whatever it contains — also when a
    *different, newer tag* is attempted (a new tag does not launder the leftovers of the previous
    one).
  - Clear both only on `DEPLOYED` or `NO_CHANGE`.

**The attempt marker.** In step 6, before the pull, the run writes `FAIL_TAG := target`,
`FAIL_COUNT` +1 (1 for a new tag), `SNAP_TAKEN` and `SNAP_INACTIVE` (if not yet taken) and
`LAST_ERROR := 'attempt in progress'`, then commits. On success these are cleared. If the process
dies mid-pull (short dump, cancelled job, killed work process), the marker is already committed:
the next run sees `FAIL_TAG` = target, treats it as a retry, and still has the original inactive
baseline. Without this, a crash would look like a fresh attempt whose snapshot already contains
the leftovers.

### 5.2 Unattended pull policy

The pull is `deserialize_checks( )` then `deserialize( checks, log )`. In dialog a person answers
what `deserialize_checks` reports; in batch nobody can. The policy **[pending]**:

- **Overwrite decisions:** never auto-overwrite an object that has local changes. If
  `deserialize_checks` reports objects to overwrite, do not pull; fail with `LOCAL_CHANGES` and
  list the objects. Rationale: this is a development system and other developers' uncommitted
  work is there.
- **Any other required decision** (package, requirements, transport, warnings that need an
  answer): do not pull; fail with `NEEDS_DECISION` and name what is missing.
- **Nothing to deserialise — only on a clean slate.** "Clean slate" means no attempt has happened
  since the last verified deployment: `FAIL_TAG` is empty **and** `SNAP_TAKEN` is not set. Then, if
  abapGit's repository status shows no difference between the remote files at the target tag and
  the local objects, no pull is issued and the result is `NO_CHANGE` (subject to checks 2–4).
- **Otherwise the pull is always issued (`force_pull`).** This covers a retry of the same tag
  **and a newer tag after a failed or crashed one**: the failed attempt's object errors are not
  known any more, so an empty status would make check 3 vacuous and could launder a partly
  deserialised system into `NO_CHANGE`.
- **The two rules above are subordinate to the decision checks:** if `deserialize_checks` reports
  `LOCAL_CHANGES` or `NEEDS_DECISION`, no pull is issued even when `force_pull` is set.
- Which `deserialize_checks` entries count as "local changes" (a changed object, a type or package
  mismatch, or every object) is defined by spike item 9.6. Until it is, the predicate is: the
  object was changed locally since abapGit last deserialised it. If the real list is broader, the
  policy would block every pull, and the spike must catch that.

### 5.3 The checks

The tag is recorded as deployed only if the verifier says `DEPLOYED` or `NO_CHANGE`, and all of the
checks that apply hold. `NO_CHANGE` still requires checks 2, 3 and 4.

1. **Timestamp moved** (applies whenever a pull was issued): `deserialized_at` differs from the
   value read immediately before this pull. If not, `FAILED` (`TIMESTAMP`) — this is the silent
   no-op. Whether a legitimate pull that changes nothing also leaves the timestamp untouched is
   spike item 9.4; if it does, a retry of an already-complete tag stays `FAILED` until the cap
   in 6.1 stops it. That is a safe failure (no false success) and is documented, not hidden.
2. **Requested ref is checked out:** the repository's selected ref equals `refs/tags/<tag>` and its
   remote commit equals the commit the tag was resolved to in step 4. Otherwise `FAILED` (`REF`).
   A tag that moves between steps 4 and 7 also fails here and is retried next run.
3. **No object-level errors:** the pull log contains no error for any object and `deploy` did not
   raise. Otherwise `FAILED` (`OBJECT_ERRORS`), listing the objects.
4. **No new inactive objects:** no object is inactive after the pull that was not inactive in the
   snapshot. Objects inactive before the snapshot (other developers' work) are ignored. Otherwise
   `FAILED` (`INACTIVE`).

`NO_CHANGE` is only possible on a clean slate (5.2): once any attempt has happened since the last
verified deployment, every pull is forced, so the outcome is `DEPLOYED` or `FAILED`, never
`NO_CHANGE`.

### Side effect on the repository's selected ref

After a pull the repository stays selected on the tag, so a later manual pull from the abapGit UI
pulls the tag, not the branch. This is intended and documented; the tool never restores the branch.

## 6. Failure handling

### 6.1 Failure classes

| Failure | Behaviour |
|---|---|
| Remote unreachable, TLS error | `FAILED` (class `REMOTE`); no tag known, so `FAIL_TAG` unchanged; `LAST_ERROR` updated; alert de-duplication applies (6.1a). |
| Authentication rejected, or `CRED_ID` set but no secure-store entry | `FAILED` (class `AUTH`); message names `CRED_ID`; as above. |
| `REPO_KEY` unknown, repository offline or without URL; stored `D`/`B` no longer parses | `FAILED` (class `CONFIG`); as above. |
| Config row inactive | Skipped silently. |
| No tag matches, or none parse | `NOTHING_TO_DO` (quiet); skipped tags appear as a note (2.2). |
| `LOCAL_CHANGES` or `NEEDS_DECISION` (5.2) | `FAILED` with the failed check; `FAIL_TAG` set; the ref may already be on the tag. |
| Switch succeeds, pull raises | `FAILED`; `FAIL_TAG` set; the next run repeats (5.1). |
| Pull "succeeds", verifier fails | `FAILED` with the failed check; as above. |
| `FAIL_COUNT` reached `MAX_ATTEMPTS` for the target tag | Not pulled again. The attempt whose failure **made** `FAIL_COUNT` reach the cap logs "attempt limit reached" once and sets the error exit status once (it is that attempt's own failure). Every later poll is **quiet**: no log entry, no error status, only the heartbeat — `LAST_ERROR` keeps the real error and the status report shows the repository red (6.6). Stays so until a different tag becomes the target, or an operator intervenes (6.5). Bounds repeated partial pulls into a shared development system. **[pending]** |
| `FAIL_COUNT` below the cap, but the back-off has not elapsed | `NOTHING_TO_DO` (quiet), "waiting for back-off" (6.1a). |
| Lock not obtained | `SKIPPED`; nothing written. |
| Locked objects, unusable transport request, missing authorisation | `FAILED`; cause logged; `FAIL_TAG` set. |
| Job cancelled, killed, or dumped mid-pull | The attempt marker (5.1) is already committed, so the next run retries against the original baseline. The enqueue lock is released when the session ends. A dump inside abapGit ends the whole job, so repositories after it are handled next run. |

### 6.1a Back-off and alert de-duplication

At one poll per minute, retries and repeated failures would otherwise be hammered and flood the log
and the job overview.

- **Back-off between attempts** of the same tag: after attempt *n* fails (or crashes), the next
  attempt is not made before `LAST_ERROR_AT` plus a wait of 1, 2, 5, 10 minutes for *n* = 1, 2, 3, 4
  and 10 minutes for any later *n*. **[pending]** With the default `MAX_ATTEMPTS` of 5, a transient
  cause has about 18 minutes to clear before an operator is needed. A run that finds the back-off
  not yet elapsed is `NOTHING_TO_DO` (quiet).
- **Alert de-duplication.** A failure is *logged and sets the error exit status* (6.4) only if it is
  new: its `LAST_ERROR_CLASS` differs from the recorded one, or at least 60 minutes have passed
  since `LAST_ALERT_AT`. An identical repeat only updates the heartbeat. So a git-host outage
  produces one log entry and one cancelled job, then one reminder per hour, and not one per minute.
- A poll that reaches the remote successfully **clears** a recorded `REMOTE`, `AUTH` or `CONFIG`
  error (`LAST_ERROR*` and the class), so recovery is visible; an `ATTEMPT` error is cleared only
  by a verified deployment (6.2).

### 6.2 What each outcome writes to the state row

| Outcome | Written | Not touched |
|---|---|---|
| row-3 prefix (2.2: `DEPLOY_ON_FIRST_RUN`) | `BASELINED` := X, `BASELINE_TAG` := empty — **written together with the fields of the outcome that follows** (attempt marker or `NOTHING_TO_DO`) | – |
| `BASELINED` | `BASELINED`, `BASELINE_TAG` | everything else |
| `NOTHING_TO_DO` (including waiting for back-off, and a capped tag on later polls) | the heartbeat (below), `LATEST_TAG`, `LAST_NOTE_HASH` if the notes changed, plus the row-3 prefix if it applies | everything else |
| `SKIPPED` (lock busy) | **nothing** | everything |
| `DRY_RUN` | **nothing**, not even a baseline or the heartbeat | everything |
| attempt starts (step 6) | `FAIL_TAG`, `FAIL_COUNT`+1 (1 for a new tag), `SNAP_TAKEN`, `SNAP_INACTIVE` if not yet taken, `LAST_ERROR := attempt in progress` | `DEPLOYED_*`, `BASELINE_*` |
| `DEPLOYED`, `NO_CHANGE` | `DEPLOYED_TAG/COMMIT/AT`, `LAST_OUTCOME`; clear `FAIL_*`, `SNAP_*`, `LAST_ERROR*` | `BASELINE_*` |
| `FAILED` after an attempt started | `LAST_ERROR*` := the actual error, class `ATTEMPT`, `LAST_OUTCOME`; `LAST_ALERT_AT` if it alerts (6.1a) | `DEPLOYED_*`, `BASELINE_*`, `FAIL_*`, `SNAP_*` |
| `FAILED` without a target tag or before the marker was written (`REMOTE`, `AUTH`, `CONFIG`) | `LAST_ERROR*`, its class, `LAST_OUTCOME`, `LAST_ALERT_AT` if it alerts | everything else |

**Heartbeat.** Every path through step 9 also writes `LAST_POLL_AT`, and `LAST_POLL_OK_AT` whenever
tag listing in step 4 succeeded. `SKIPPED`, inactive rows and `DRY_RUN` do not write it. This is
the only write a quiet poll makes besides `LATEST_TAG` and the note hash, so it is a single-row
update.

"Deployed fields unchanged" is what "state unchanged" means throughout: a failed attempt never
changes `DEPLOYED_*`. There is no push alerting (mail, chat) in v1; consumers read the outward
status, the status report, the job overview, the application log, `FAIL_COUNT` and `LAST_ERROR`
(6.6).

### 6.3 Locking and rollback

The lock object `EZ_CDEPLOY` locks `Z_CDEPLOY_STATE` by `REPO_KEY`. It is requested **before**
step 2 with `_SCOPE = 1` and `_WAIT = space` (a busy lock returns immediately), and released only
in step 10. Scope 1 is required: with the default scope 2 the lock would be released at the first
`COMMIT WORK`, which abapGit's deserialisation performs. The lock's purpose is to stop two
overlapping runs of this tool. With a one-minute period, overlap is the **normal** case whenever a
pull takes longer than a minute, not an edge case: the later instance finds the lock busy and ends
that repository as `SKIPPED`, silently and without error. Locks are per repository, so two
instances may pull two *different* repositories at the same time; that is allowed. The lock does
**not** serialise against a person pulling the same repository in abapGit's UI at the same time.
That case may succeed or fail; the verifier cannot tell whose pull moved the timestamp, and this is
a documented limitation.

**With a one-minute period, the lock is load-bearing, so spike item 9.9 is a go/no-go for the
one-minute period.** If anything during deserialisation can release the scope-1 lock early, the
next instance would find `FAIL_TAG` set, treat it as a retry and start a second concurrent pull of
the same repository. In that case one of two fallbacks is required before the period may stay at one
minute: a second guard that refuses to start while an earlier instance of this job is still active
in the job tables, or a longer period than the longest expected pull.

No automatic rollback. If a tag fails part-way the system may be partly updated; recovery is the
next tag, or re-running once the cause is fixed.

### 6.4 Reporting and exit status

One application-log entry per repository per run **for every outcome except `NOTHING_TO_DO` and
`SKIPPED`**: run outcome, tag, failed check if any. Repeated identical failures are logged only as
6.1a allows, and notes only when they change (2.2). A minute-by-minute job that logged "nothing to
do" would bury the entries that matter. The spool likewise lists only what was logged, and is empty
for a quiet poll. State and
log are written and committed **per repository** (step 9). At the end the report prints the summary
to the spool and then decides the exit status; nothing that matters is left uncommitted when the
final message aborts the job.

The report ends with message type `E` if **any** repository ended `FAILED` *and that failure
alerts under 6.1a* (a new failure, or a reminder after an hour), so the job appears as cancelled in
the job overview once per problem and not once per minute. Runs with only `DEPLOYED`, `NO_CHANGE`, `NOTHING_TO_DO`,
`BASELINED`, `SKIPPED` end normally. `dry-run` never sets the error status and never calls the
deployer, but does contact the remote, read credentials and write log entries marked `DRY-RUN`. If
a dry run meets a failure that a real run would hit (remote unreachable, bad credential, attempt
limit reached), it logs "would fail: …" and, by the rule above, still ends normally: **for
`dry-run`, "never sets the error status" wins over "E if any repository failed".**

### 6.5 Operator recovery

Automatic recovery stops at the attempt cap by design. To resume a repository, an operator:

1. Reads `LAST_ERROR` and the application log for the cause, and fixes it (for example activates
   or removes objects the failed attempt left inactive, or resolves the local changes).
2. **Resets the attempt count** (`FAIL_COUNT` := 0) with the setup report's `RESET_ATTEMPTS`
   function. `FAIL_TAG` and the stored inactive baseline are kept, so the next attempt is still
   judged against the original baseline and cannot record a false success.
3. If check 4 keeps failing only because *other developers* have made objects in the package
   inactive since the baseline was taken, and the operator has verified by hand that nothing left
   inactive by the failed attempt remains, the setup report's `CLEAR_BASELINE` function clears
   `FAIL_TAG`, `SNAP_TAKEN` and `SNAP_INACTIVE`. **This deliberately re-opens the false-success
   risk for the next attempt** and is why it is a separate, explicit action.

4. **A stray high tag** (for example someone pushes `v9.0.0-test`, which becomes `D` and makes every
   real release below it "older"): delete the tag in git, then run the setup report's
   `RESET_DEPLOYED`, which clears `DEPLOYED_TAG`, `DEPLOYED_COMMIT` and `DEPLOYED_AT`. The next
   poll then follows rows 8 or 9 of the decision table and deploys the newest eligible tag.

A complete but unrecorded deserialisation (a crash after the last object, before the outcome was
written) is recovered the same way; whether the retry can pass by itself depends on spike item 9.4.

### 6.6 Monitoring from outside

Requirement: someone **without SAP access** can tell which deployments succeeded and which failed,
and whether the job itself is alive. A dead job produces no failures at all, so liveness is part of
monitoring. The layers, from most to least accessible:

1. **Outward status on GitHub (`REPORTER`).** For a repository with `ENVIRONMENT` and
   `REPORT_CRED_ID` set, the tool publishes to the repository's *Deployments* API: at the attempt
   marker it creates a deployment for the tag under the configured environment and sets it
   `in_progress`; after the verdict it sets `success`, or `failure`. The repository's Environments
   and Deployments pages then show, for every system, which tag it runs and whether the last
   attempt worked, with no inbound path to SAP and no SAP login.
   - **Everything reported is as public as the repository.** On a public repository the
     Deployments pages, the environment names and the account that owns the reporting PAT are
     readable by anyone. "No secret is sent" is not the same as "nothing private is published".
     Therefore: `ENVIRONMENT` must be a neutral alias (4.1); the PAT belongs to a **technical
     account**, not a person; and reporting to a public repository stays **off unless
     `REPORT_ON_PUBLIC` is set**. The reporter reads the repository's visibility from the API
     before its first call in a run and stays silent (with a note, 2.2) if it is public and the
     flag is off.
   - **The payload is fixed and minimal:** `ref` = the tag, the environment alias, and as
     description only the `failed_check` value (an enumeration such as `INACTIVE`), or empty. Never
     `LAST_ERROR`, object names, package names, host names or messages. No `log_url`,
     `environment_url` or `payload` field is set.
   - **API details.** Create deployment with `auto_merge` = false, `required_contexts` = an empty
     list (otherwise GitHub refuses with 409 when the tag's commit has red checks) and
     `production_environment` = false; then post statuses. Creating a deployment fires any
     `deployment`-event workflow in the repository; that is documented in the README. The reporting
     PAT is fine-grained with one permission on one repository: *Deployments: read and write*.
   - **Stateless and crash-safe.** `started` first looks up existing deployments for this
     environment and marks any still `in_progress` or `queued` as `error` ("superseded"), then
     creates the new one. `finished` looks up the latest deployment for this environment and tag and
     posts the verdict to it; if there is none (because `started` failed), it creates one first. A
     crash after `started` therefore leaves at most one `in_progress` deployment, closed by the next
     attempt, and no deployment id has to be stored.
   - **Best effort with a bound.** Every call has a timeout of 5 seconds. A reporting failure
     (network, token, rate limit) is logged and never fails or changes a deployment. `started`
     makes at most four calls and `finished` at most three, so a repository is delayed by at most
     seven times the timeout, and only on a real attempt: quiet polls make no reporter call at all
     (step 9).
   - **GitHub (`github.com`) only.** The host is recognised from the repository URL; GitHub
     Enterprise and other hosts are out of scope for v1 and reporting is off for them.
     **[pending]**
   - **Separate PAT.** Writing deployments needs more than read access; it uses `REPORT_CRED_ID`
     through `CREDENTIALS->get_report_token`, never through abapGit's login manager, and not the
     read-only pull PAT (4.3).
2. **Heartbeat.** `LAST_POLL_AT` and `LAST_POLL_OK_AT` (4.2). A monitor that sees `LAST_POLL_AT`
   stale knows the job is not running; one that sees only `LAST_POLL_OK_AT` stale knows git is
   unreachable. Both are shown in layer 3.
3. **Status report `Z_CONTINUOUS_DEPLOYMENT_STATUS`.** A read-only list built **from the state
   table alone** (no remote call, no credential), one line per repository: deployed tag, commit and
   time (UTC), `LATEST_TAG`, `LAST_OUTCOME`, `FAIL_TAG` and `FAIL_COUNT`, `LAST_ERROR` and its
   class, the current notes (2.2), both heartbeats, and a traffic-light column:
   - **green:** `DEPLOYED_TAG` equals `LATEST_TAG`, no recorded failure, heartbeat fresh;
   - **yellow:** an attempt in progress and younger than the in-progress threshold, or a baselined
     repository with nothing deployed yet, or `LATEST_TAG` newer than `DEPLOYED_TAG` with no
     failure (a deployment is due);
   - **red:** failing or capped, `AUTH` / `REMOTE` / `CONFIG` error, heartbeat stale, or an attempt
     "in progress" **older than the threshold** (that is what a crashed attempt looks like).

   Thresholds are report parameters: heartbeat stale after **3 minutes** by default, attempt in
   progress stale after **30 minutes**. Runnable in the GUI and readable through ADT.
4. **SAP job monitoring.** The job ends cancelled when any repository failed (6.4), so standard job
   monitoring and any alerting built on it already fires.
5. **Application log `ZCDEPLOY`** for detail (SLG1). Only non-trivial outcomes are logged (6.4).

The thresholds live in the status report, not in state.

## 7. Testing

**Unit tests** (ABAP Unit; no network, no abapGit; fakes for every interface):

- decision table 2.2, every row **in order**, including first run with and without
  `DEPLOY_ON_FIRST_RUN`, baselining with no tags then a first release, the steady state after
  baselining (row 10), a failed first deploy under `DEPLOY_ON_FIRST_RUN` being retried, tag
  re-pointed, older tag
- tag grammar (2.1): `v1.2.3`, `v1.2.3-rc1`, `v1.2.3-anything` eligible; `1.2.3`, `V1.2.3`,
  `v1.2`, `v1.2.3.4`, `v1.2.3rc1`, `v1.2.3-`, `v01.2.3`, `release-1.2.0`,
  `my-customer-final17-2026` skipped; `v1.10.0` beats `v1.9.0`; `v1.2.0` beats `v1.2.0-rc1`;
  `-rc.10` beats `-rc.2` but `-rc10` does not beat `-rc2`; a numeric identifier is older than an
  alphanumeric one; a component above 2147483647 is skipped; equality of `T` and `D` is by parsed
  version and suffix
- heartbeat: a quiet poll writes only `LAST_POLL_AT` / `LAST_POLL_OK_AT`, no log entry, empty
  spool; an unreachable remote updates `LAST_POLL_AT` but not `LAST_POLL_OK_AT`
- reporter: called only in a run that reached step 6 (never for quiet polls, `BASELINED`, the cap,
  or failures before the marker); `started` is called after the commit; `finished` is called in
  every run that called `started`, and creates the deployment itself if `started` failed; a
  reporter that raises,
  times out or errors never changes the outcome; the payload contains only tag, alias and
  `failed_check`, never `LAST_ERROR`, object or package names; no reporter call and a note when the
  repository is public and `REPORT_ON_PUBLIC` is off; `ENVIRONMENT` without `REPORT_CRED_ID` (or
  the reverse) means reporting off with a note; a leftover `in_progress` deployment is marked
  `error` by the next `started`
- the report PAT is never passed to the login manager, and never appears in any output
- credentials: only a PAT can be entered; the basic-auth user-name part is the fixed label
- every stored or printed timestamp is UTC `TIMESTAMPL` (the bare names are also forbidden by
  abaplint, 10)
- back-off: after attempt *n* the next attempt waits 1, 2, 5, 10, 10 … minutes from
  `LAST_ERROR_AT`; a run inside the wait is `NOTHING_TO_DO`; a crash waits the same way
- alert de-duplication: an identical repeated failure neither logs nor sets the error status for
  60 minutes; a different class does at once; a reminder follows after 60 minutes; a successful
  poll clears a recorded `REMOTE` / `AUTH` / `CONFIG` error
- notes are logged only when their hash changes
- the status report colours: crashed attempt older than the threshold is red; a stale heartbeat is
  red; a due deployment is yellow; it uses no remote call
- each verifier check fails on its own and yields the right `failed_check`
- `NO_CHANGE` requires checks 2–4, is only possible on a clean slate, and only the verifier
  produces it
- **retry after partial failure:** first attempt fails, second attempt cannot pass on leftovers
  (check 4 uses the stored baseline) and cannot pass on a timestamp the first attempt already
  moved (check 1 reads it fresh)
- **zero-inactive baseline:** the package has *no* inactive objects at the first attempt, the
  attempt leaves some, the retry must be `FAILED` (`INACTIVE`) — `SNAP_TAKEN`, not emptiness of
  `SNAP_INACTIVE`, decides that a baseline exists
- **crash path:** a fake that dies after the attempt marker is committed leaves `FAIL_TAG` set, so
  the next run is a retry with the original `SNAP_INACTIVE`, never a fresh attempt
- **a newer tag after a failed one:** the baseline is kept, `FAIL_TAG`/`FAIL_COUNT` restart, leftovers
  of the earlier tag are still reported by check 4, and the pull is **forced** even when the
  status is empty (`force_pull`), so the result cannot be `NO_CHANGE`
- `LOCAL_CHANGES` / `NEEDS_DECISION` on a forced pull → no pull issued
- attempt cap: the attempt that makes `FAIL_COUNT` reach `MAX_ATTEMPTS` logs "attempt limit
  reached" once and alerts once; every later poll makes no deployer call, writes only the
  heartbeat, logs nothing and sets no error status (`LAST_ERROR` unchanged); `MAX_ATTEMPTS` 0
  behaves as 5
- terminal outcomes (`BASELINED`, `NOTHING_TO_DO`, cap) pass through step 9 and are committed
  before the next repository is processed
- a repository with `FAIL_TAG` set is processed after those without
- a withdrawn failed tag adds the "partly at `FAIL_TAG`" note (logged once)
- `RESET_ATTEMPTS`, `CLEAR_BASELINE` and `RESET_DEPLOYED` each change exactly the fields 6.5 names;
  after `RESET_DEPLOYED` the next poll deploys the newest eligible tag (a stray `v9.0.0-test`
  scenario)
- a stored `D` or `B` that no longer parses → `FAILED` (`CONFIG`)
- `SKIPPED` and `DRY_RUN` write nothing; `NOTHING_TO_DO` writes only heartbeat, `LATEST_TAG` and
  the note hash
- state writes match the table in 6.2 for every outcome
- unattended policy: overwrite required → `LOCAL_CHANGES`, no pull issued; other decision →
  `NEEDS_DECISION`
- lock not obtained → `SKIPPED`; `ACTIVE` off → skipped
- one repository throws → the others still run; exit status is error iff any failed
- `dry-run` → deployer not called, no state written (not even a baseline; logged as "would
  baseline"), entries marked
- state and log are committed per repository: a fake that dumps on repository N leaves 1..N-1
  recorded
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
  judged against the stored baseline
- the scope-1 lock is still held after the pull has finished (see spike item 9.9)
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
4. Whether `deserialized_at` moves on a pull that deserialised something and whether it also moves
   on a legitimate pull that changes nothing. This decides whether a retry of an already-complete
   tag can ever succeed (5.3 check 1).
5. Which errors abapGit raises as exceptions and which it puts into the log; check 3 must cover
   both.
6. What `deserialize_checks` reports for unattended runs, and how to detect "decisions missing".
7. Whether changing the selected ref to a tag is honoured when the repository object was built
   earlier in the same process.
8. Release status of the `SECSTORE_*` function modules for customer use.
9. **Go/no-go for the one-minute period.** Whether anything abapGit or SAP calls during
   deserialisation releases the scope-1 lock early (for example a function that dequeues every lock
   of the session). The whole overlap protection in 6.3 depends on it; if it can be released, 6.3
   names the two fallbacks.
10. Whether GitHub accepts a PAT as the basic-authentication password with an arbitrary fixed user
    name, through abapGit's login manager and HTTP client. (GitHub only; other hosts are out of
    scope.)
11. Whether the Deployments API can be called from ABAP through the same proxy and certificate
    setup as the git requests, with a 5 second timeout, that a fine-grained PAT with only
    *Deployments: read and write* suffices for the calls in 6.6, and that visibility can be read
    with the same PAT.
12. How long a poll takes on a real system, to confirm that a one-minute period is cheap and that
    the 5–10 minute budget is realistic end to end. GitHub's tag advertisement lists **every** ref
    of the repository, including `refs/pull/*`, so it is not necessarily small on a busy repository;
    measure it, and check whether abapGit's transport can restrict it to `refs/tags/*`.

## 10. Repository layout

The repository is itself the abapGit repository for package `Z_CONTINUOUS_DEPLOYMENT`. ABAP source
lives under `src/` in abapGit format, folder logic `PREFIX`, generated by SAP's own serialisation
and never hand-written. Changes land through pull requests.

- **Squash merge only.**
- **AI agents must run at least one round of review with a separate review agent before marking a
  pull request ready for review.** A pull request an agent opened stays a draft until then.
- abaplint runs in CI and **fails the job when it finds issues**. Beyond the standard rules it
  forbids the bare names `DATUM`, `UZEIT`, `DATLO` and `TIMLO` (timestamps are UTC, 4.2), and its
  `object_naming` rule enforces the naming scheme and the DDIC length limits per object type
  (3.2). abaplint cannot see a package's real name, so the package name is checked in review.
