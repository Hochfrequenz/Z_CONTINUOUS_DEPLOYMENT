# Continuous deployment into SAP development systems via abapGit — design

Status: draft v2 for review. Date: 2026-09-28.

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
- **abapGit is the engine.** The tool orchestrates abapGit; it does not re-implement git or
  object deserialisation.
- **Z-namespace tool.** All objects of *this tool* live in the customer namespace. The repositories
  it deploys may contain any objects abapGit can deserialise, including namespaced ones; that is
  abapGit's concern, not this tool's.
- **Public and private repositories.** "No credential" is a valid configuration.
- **Development systems.** Promotion to later systems uses the normal transport route and is out
  of scope.
- **Older releases stay in scope [pending].** The tool must work on an ABAP release with an older
  abapGit as long as the APIs in section 8 exist there.

### Non-goals (v1)

Automatic rollback, a UI, webhooks or any inbound endpoint, alerting (mail, chat), deployment to
non-development systems, deserialisation without abapGit, a history of past deployments.

## 2. Behaviour

A batch job runs the report periodically (suggested: every 15 minutes). For each active
configured repository:

1. Resolve the repository by key. Unknown key or offline repository → failure (section 6).
2. Ask the remote for its tags (`TAG_SOURCE`). No network call is repeated when nothing can have
   changed; there is no local cache in v1.
3. Choose the target tag (section 2.1).
4. Decide (section 2.2). If nothing to do, log and continue.
5. `dry-run`: log the decision and stop.
6. Take a **snapshot** (section 5) — the verifier's "before" state.
7. Acquire the repository lock. Not obtained → skip, log.
8. Switch the repository to the tag and pull (`DEPLOYER`).
9. Verify against the snapshot (`VERIFIER`).
10. If verified: write state. Otherwise: record the failure, leave the deployed tag unchanged.
11. Release the lock, write the log.

A failure in one repository never stops the others. At the end the report sets its exit status
(section 6.4).

The trigger is a **git tag**. Branch HEAD is deliberately not the trigger: a tag is a deliberate
promotion step and a clean rollback target.

### 2.1 Selecting the target tag

- `TAG_PATTERN` uses ABAP `CP` pattern syntax (`*` any string, `+` any character), matched against
  the tag name without the `refs/tags/` prefix. Example: `v*`.
- A matching tag must additionally parse as a version: an optional prefix, then
  `MAJOR.MINOR.PATCH` (numbers), then an optional pre-release suffix. Tags that match the pattern
  but do not parse are **skipped and logged**; they are never an error for the whole repository.
- Pre-releases (a suffix such as `-rc1`) are **not eligible** in v1.
- Ordering is numeric per component (`v1.10.0` is newer than `v1.9.0`).
- Annotated and lightweight tags are both accepted; the tag is resolved to the commit it points to
  (annotated tags carry a separate tag-object hash, which must not be mistaken for the commit).
- No tag matches → "nothing to deploy", not an error, but logged.

### 2.2 Deciding whether to deploy

- **No state row (first run) [pending]:** record the newest eligible tag as the **baseline**
  without deploying it. Reason: the repository may already have been pulled manually from a branch,
  and silently moving it to a tag is surprising. A per-repository flag `DEPLOY_ON_FIRST_RUN`
  (default off) overrides this.
- **Target newer than deployed tag:** deploy.
- **Target equal to deployed tag:** nothing to do.
- **Target older than deployed tag** (a tag was deleted or withdrawn): nothing to do, logged as a
  warning. Comparison is by version order, never by commit ancestry. Going backwards is a manual
  action.
- **Repository already on the target ref but state not updated** (a previous run switched but
  failed): treated as "deploy" — the pull is repeated. The snapshot is taken again, so the
  verifier's baseline is correct.

## 3. Components

Each unit has one purpose and is used through an interface, so the orchestrator can be tested with
fakes.

| Unit | Responsibility | Depends on |
|---|---|---|
| `ZIF_CDEPLOY_CONFIG` | Return the active repository configurations | config table |
| `ZIF_CDEPLOY_TAG_SOURCE` | Return the eligible tags of a repository, resolved to commits | abapGit git transport |
| `ZIF_CDEPLOY_STATE` | Read/write deployed-tag state and failure counters | state table |
| `ZIF_CDEPLOY_DEPLOYER` | Switch to a tag and pull | abapGit repository API |
| `ZIF_CDEPLOY_VERIFIER` | Take a snapshot; judge the result of a pull | abapGit repository object |
| `ZIF_CDEPLOY_LOG` | Write to the application log and return a summary | application log |
| `ZCL_CDEPLOY_RUN` | Orchestrate section 2 | all interfaces above |
| `Z_CONTINUOUS_DEPLOYMENT` (report) | Batch entry point; parameter `dry-run` | `ZCL_CDEPLOY_RUN` |
| `Z_CONTINUOUS_DEPLOYMENT_SETUP` (report) | Store/rotate/delete a credential | secure store |

Production implementations: `ZCL_CDEPLOY_CONFIG_DB`, `ZCL_CDEPLOY_TAGS_ABAPGIT`,
`ZCL_CDEPLOY_STATE_DB`, `ZCL_CDEPLOY_DEPLOYER_ABAPGIT`, `ZCL_CDEPLOY_VERIFIER`,
`ZCL_CDEPLOY_LOG_BAL`.

### 3.1 Interface contracts (informative; exact signatures belong in the plan)

| Method | Input | Output |
|---|---|---|
| `CONFIG->get_active` | – | table of config rows |
| `TAG_SOURCE->get_tags` | repo key, tag pattern | table of `{name, version, commit}`, eligible only |
| `STATE->get` / `->save_success` / `->save_failure` | repo key (and tag/commit/error) | state row |
| `VERIFIER->snapshot` | repo key | `{deserialized_at, commit, inactive_objects}` |
| `VERIFIER->verify` | repo key, target tag/commit, snapshot | `{ok, outcome, failed_check}` |
| `DEPLOYER->deploy` | repo key, tag | `{messages, object_errors}` |
| `LOG->add` / `->summary` | repo key, outcome | – / printable text |

`outcome` is one of `DEPLOYED`, `NO_CHANGE`, `FAILED`. `failed_check` is one of `TIMESTAMP`, `REF`,
`INACTIVE`, `OBJECT_ERRORS`, or initial.

### 3.2 Naming

Package `Z_CONTINUOUS_DEPLOYMENT`; tables `Z_CDEPLOY_REPO`, `Z_CDEPLOY_STATE`; lock object
`EZ_CDEPLOY`; application log object `ZCDEPLOY`, subobject `RUN`. Object prefix `CDEPLOY` keeps
names inside DDIC limits (tables 16, classes and interfaces 30, lock objects 16 characters).

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

- The repository must already be registered in abapGit. URL, package and branch live in abapGit.
- Repositories are addressed by **key, never by a name substring**.
- Keys are assigned per system by abapGit, so rows are created per system and are not transported.

**Transport request [pending]:** the tool does not keep its own request. It uses the repository's
own abapGit setting `transport_request`, maintained in abapGit's UI. If the setting is empty and
the package is transportable, the pull fails with a clear message. Rotating a released request is
therefore an abapGit-settings change, documented in the README. The batch user must be allowed to
add objects to that request.

### 4.2 `Z_CDEPLOY_STATE` — one row per repository

| Field | Meaning |
|---|---|
| `REPO_KEY` | Key. |
| `DEPLOYED_TAG` | Last tag whose deployment was verified. |
| `DEPLOYED_COMMIT` | Its commit. |
| `DEPLOYED_AT` | Timestamp of the verified deployment. |
| `FAIL_TAG` | Tag of the current failing attempt, else initial. |
| `FAIL_COUNT` | Consecutive failures for `FAIL_TAG`. Reset on success or on a different tag. |
| `LAST_ERROR` | Last failure text (never contains a credential). |
| `LAST_ERROR_AT` | Timestamp of it. |

State holds the **current** deployed tag only, not a history.

### 4.3 Credentials for private repositories

- abapGit normally asks for credentials in a dialog and keeps them in session memory only. A batch
  job has no dialog, so the deployer supplies them: before a fetch it reads the token and hands it
  to abapGit's login manager for the repository URL.
- Tokens are kept in SAP's secure store (`SECSTORE`). `CRED_ID` is only the key. The store's
  access list restricts reading to this tool's own class. A token is never written to a table,
  the log, the state table, or a message.
- Recommended token: fine-grained, read-only, limited to the single repository.
- `Z_CONTINUOUS_DEPLOYMENT_SETUP` stores, rotates and deletes tokens. It never displays one.

### 4.4 Network prerequisites (documented, not implemented)

Proxy settings are inherited from abapGit. The git host's certificate chain must be trusted in the
system's certificate store. The batch user needs authorisation for abapGit deserialisation and the
repository's transport request.

## 5. Verification

The verifier is called twice per deployment: `snapshot` **before** the pull and `verify` after.
After a pull, the tag is recorded as deployed only if all four checks hold.

1. **Timestamp moved.** The repository's `deserialized_at` differs from the snapshot value. The
   outcome `NO_CHANGE` is distinct from failure and is decided as follows: if the repository's
   remote commit equals the snapshot's deserialised commit *and* the timestamp did not move, the
   pull was legitimately empty → `NO_CHANGE`, state **is** written (the tag is deployed), and
   nothing retries. If the commit differs but the timestamp did not move → `FAILED` (`TIMESTAMP`).
2. **Requested ref is checked out.** The repository's selected ref equals `refs/tags/<tag>` and its
   remote commit equals the commit the tag resolves to. Otherwise `FAILED` (`REF`).
3. **No object-level errors.** The log returned by the pull contains no error for any object.
   Otherwise `FAILED` (`OBJECT_ERRORS`), listing the objects.
4. **No new inactive objects.** Inactive objects that appear in the repository's objects after the
   pull and were not inactive in the snapshot. Pre-existing inactive objects from other developers
   are ignored. Otherwise `FAILED` (`INACTIVE`).

Check 1 catches a pull that reports success and changes nothing (a stale session, for example).
Check 2 catches the wrong ref or repository. Checks 3 and 4 catch partial deserialisation.

### Side effect on the repository's selected ref

After a pull the repository stays selected on the tag, so a later manual pull from the abapGit UI
pulls the tag, not the branch. This is intended and documented. The tool never restores the
previous branch.

## 6. Failure handling

### 6.1 Failure classes

| Failure | Behaviour |
|---|---|
| Remote unreachable, authentication or TLS error | Log; state unchanged; retry next run. |
| `CRED_ID` set but no secure-store entry | Fail this repository; message names `CRED_ID`. |
| `REPO_KEY` unknown, repository offline or without URL | Fail this repository. |
| Config row invalid (bad pattern, inactive) | Invalid → fail with message; inactive → skip silently. |
| No tag matches / no tag parses | Not an error; logged; nothing done. |
| Switch to tag succeeds, pull raises an error | Log the error; state unchanged. The next run repeats the pull (2.2). |
| Pull "succeeds", verifier fails | Failed deployment; log which check; state unchanged. |
| Object-level errors in the pull log | As above (`OBJECT_ERRORS`). |
| Lock not obtained | Skip this repository; log. |
| Locked objects, missing or unusable transport request, missing authorisation | Fail this repository; log the cause. |
| Runtime limit reached | Job terminates; unprocessed repositories are picked up next run. |

### 6.2 Retry and noise

State moves only after verification, so a failed tag is retried each run. `FAIL_COUNT` is kept in
the state table and shown in the log; there is no alerting in v1 (the maintainer can filter the
batch job's spool or the application log).

### 6.3 Rollback and locking

No automatic rollback. If a tag fails part-way the system may be partly updated; recovery is the
next tag, or re-running once the cause is fixed. The lock object `EZ_CDEPLOY` (lock argument: the
repository key) prevents two overlapping runs of this tool, for example when a job runs longer
than its period. It does not stop a person pulling the same repository from abapGit's UI at the
same time; that case is handled by abapGit's own object locks and shows up as a failure above.

### 6.4 Reporting and exit status

One application-log entry per repository per run: tag, outcome, failed check if any. The report
prints the same summary to the spool.

The report ends with message type `E` if **any** repository failed, so the batch job shows as
cancelled in the job overview. A run with only successes, no-changes and skips ends normally.
`dry-run` never sets the error status and never calls the deployer, but it does contact the
remote, read credentials and write application-log entries marked `DRY-RUN`; it writes no state.

## 7. Testing

**Unit tests** (ABAP Unit; no network, no abapGit; fakes for every interface):

- newer tag → deploy; state written only when the verifier passes
- same tag as state → nothing called; older tag → nothing called, warning logged
- first run without state → baseline recorded, deployer not called; with `DEPLOY_ON_FIRST_RUN` →
  deployed
- no matching tag; matching but unparseable tag skipped; pre-release ignored; `v1.10.0` beats
  `v1.9.0`
- each verifier check fails on its own and yields the right `failed_check`; a legitimately empty
  pull yields `NO_CHANGE` and writes state
- verifier fails → state unchanged, `FAIL_COUNT` increments; a different tag resets it
- `ACTIVE` off → skipped; invalid config → failed, others continue
- lock not obtained → skipped
- one repository throws → the others still run; exit status is error if any failed
- `dry-run` → deployer not called, no state written, log entries marked
- **a token never appears in any log message or state field** (asserted on all fakes' outputs)

**Integration tests** (development system, throwaway repository):

- pull a tag into a local package, and into a transportable package using the repository's
  transport request setting
- silent no-op: pull a tag that is already deployed → `NO_CHANGE`, not failure; and a deployer
  fake that reports success without pulling → verifier rejects it
- private repository with a token, public without
- two registered repositories whose names overlap: the deployer acts only on the one whose key is
  configured (observed via the repository objects touched)
- re-run after a failed pull with the repository already on the target ref → pulls again

## 8. Verified facts about abapGit (checked against installed source, not run live)

Checked on one current and one older ABAP release, both with the abapGit developer edition
installed; APIs identical on both:

- Repository keys are `c length 12` (`ZIF_ABAPGIT_PERSISTENCE`).
- The repository object exposes `ms_data-deserialized_at` and the selected ref.
- `select_branch` accepts any ref name; `refs/tags/*` is treated as a tag.
- `get_git_transport( )->branches( url )->get_tags_only( )` lists remote tags. It is commented
  "for potential future use" in abapGit, so it is not a stability contract.
- A pull is `deserialize_checks( )` then `deserialize( is_checks, ii_log )`; the log carries
  object-level messages.
- `ZCL_ABAPGIT_LOGIN_MANAGER=>set( iv_uri, iv_username, iv_password )` stores credentials in memory
  for the current process.
- The secure store is available through `SECSTORE_INSERT_ITEM`, `SECSTORE_READ_ITEM` and
  `SECSTORE_DELETE_ITEM`, with an access list restricting readers by program.
- abapGit's local settings contain a per-repository `transport_request`.

## 9. Still to verify before the plan (a spike, outcomes recorded here)

1. An actual pull of a tag through `select_branch` + `deserialize` in a batch context, including
   what the login manager needs for a private repository.
2. How `get_tags_only` represents annotated tags (tag object vs. peeled commit).
3. Whether `deserialized_at` moves when a pull deserialises nothing (drives check 1).
4. Whether changing the selected ref to a tag is honoured when abapGit was constructed earlier in
   the same process.
5. Whether the standalone abapGit variant (single program) is in scope. If it is, the tool cannot
   call global classes and needs a different adapter; **not supported in v1** unless the spike
   shows otherwise.

## 10. Repository layout

The repository is itself the abapGit repository for package `Z_CONTINUOUS_DEPLOYMENT`. ABAP source
lives under `src/` in abapGit format, folder logic `PREFIX`, generated by SAP's own serialisation
and never hand-written. abaplint runs in CI. Changes land through pull requests.
