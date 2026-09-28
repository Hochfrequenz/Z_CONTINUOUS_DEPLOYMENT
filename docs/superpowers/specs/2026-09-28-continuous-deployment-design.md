# Continuous deployment into SAP development systems via abapGit — design

Status: draft for review. Date: 2026-09-28.

## 1. Goal

Give SAP development systems the experience of continuous deployment, limited to what real SAP
landscapes allow. A background job **inside** the SAP system notices that a new version of an
abapGit-managed repository has been released in git and deploys it with an abapGit pull. The
deployment is verified, logged, and never reported as successful unless it demonstrably happened.

### Constraints that drive the design

- **Outbound-only.** Nothing may connect *in* to the SAP system. Many target systems cannot expose
  any endpoint to the internet. The only network path assumed is SAP → git host over HTTPS,
  possibly through a proxy. Systems with no outbound access at all are out of scope for v1.
- **abapGit is the engine.** The tool orchestrates abapGit; it does not re-implement git or
  object deserialisation.
- **Z-namespace.** All objects live in the customer namespace (`Z*`). No namespace registration,
  no repair licence.
- **Public and private repositories.** "No credential" is a valid configuration.
- **Development systems.** The target is a system where code is developed. Promotion to later
  systems uses the normal SAP transport route and is out of scope.

### Non-goals (v1)

Automatic rollback, a UI, webhooks or any inbound endpoint, deployment to non-development
systems, deserialisation without abapGit.

## 2. Behaviour

A batch job runs the report periodically (default suggestion: every 15 minutes). For each active
configured repository it:

1. asks the remote for its tags,
2. selects the newest tag matching the configured pattern,
3. compares it with the last successfully deployed tag,
4. if newer, switches the abapGit repository to that tag and pulls,
5. **verifies** the result,
6. records the tag as deployed **only if verification passed**,
7. logs the outcome.

The trigger is a **git tag** (or a release, which is a tag). Branch HEAD is deliberately not the
trigger: a tag is a deliberate promotion step and a clean rollback target.

A failure in one repository never stops the others.

## 3. Components

Each unit has one purpose and is used through an interface, so the orchestrator can be tested
with fakes.

| Unit | Responsibility | Depends on |
|---|---|---|
| `ZIF_CDEPLOY_CONFIG` | Return the list of watched repositories | config table |
| `ZIF_CDEPLOY_TAG_SOURCE` | Return the newest tag of a repository matching a pattern | abapGit git transport |
| `ZIF_CDEPLOY_STATE` | Get/set the last successfully deployed tag per repository | state table |
| `ZIF_CDEPLOY_DEPLOYER` | Switch the repository to a tag and pull into the configured request | abapGit repository API |
| `ZIF_CDEPLOY_VERIFIER` | Prove that a pull really happened | abapGit persistence, inactive-object check |
| `ZIF_CDEPLOY_LOG` | Write results to the application log and return a summary | application log |
| `ZCL_CDEPLOY_RUN` | Orchestrate the loop in section 2 | all interfaces above |
| `Z_CONTINUOUS_DEPLOYMENT` (report) | Batch entry point; `dry-run` parameter | `ZCL_CDEPLOY_RUN` |
| `Z_CONTINUOUS_DEPLOYMENT_SETUP` (report) | Store or rotate a credential in the secure store | secure store |

Production implementations: `ZCL_CDEPLOY_CONFIG_DB`, `ZCL_CDEPLOY_TAG_SOURCE_ABAPGIT`,
`ZCL_CDEPLOY_STATE_DB`, `ZCL_CDEPLOY_DEPLOYER_ABAPGIT`, `ZCL_CDEPLOY_VERIFIER`,
`ZCL_CDEPLOY_LOG_BAL`.

### Naming

Package `Z_CONTINUOUS_DEPLOYMENT`; tables `Z_CDEPLOY_REPO`, `Z_CDEPLOY_STATE`; application log
object `ZCDEPLOY`. Object prefix `CDEPLOY` keeps names within DDIC length limits (tables 16,
classes/interfaces 30 characters) while staying readable.

## 4. Configuration

One row per watched repository in `Z_CDEPLOY_REPO`:

| Field | Meaning |
|---|---|
| `REPO_KEY` | The abapGit repository key. Mirrors abapGit's own key type; it is the identity. |
| `TAG_PATTERN` | Which tags are deployable, e.g. `v*`. |
| `TRKORR` | Standing transport request the pull writes into. Empty for a local package. |
| `CRED_ID` | Reference to a stored credential. Empty means a public repository. |
| `ACTIVE` | Per-repository switch. |
| `DESCRIPTION` | Free text for the administrator. |

- The repository must already be registered in abapGit. URL, package and branch live in abapGit,
  not here. A `REPO_KEY` that does not exist is a per-repository error.
- Keys are assigned per system by abapGit, so configuration rows are created per system during
  setup and are not transported between systems.
- Repositories are always addressed by **key, never by a name substring**.
- Tag ordering uses version order, not string order (`v1.10.0` is newer than `v1.9.0`).

### Credentials for private repositories

- abapGit normally asks for credentials in a dialog and keeps them in session memory. A batch job
  has no dialog, so the deployer supplies them itself.
- Tokens are kept in SAP's secure store. `CRED_ID` is only the lookup key. A token is never stored
  in a plain table, never written to the log, and never put in the state table.
- Before a fetch the deployer reads the token and passes it to abapGit's login handling for that
  repository URL.
- Recommended token: fine-grained, read-only, limited to the single repository.
- `Z_CONTINUOUS_DEPLOYMENT_SETUP` stores and rotates tokens.

### Network prerequisites (documented, not implemented here)

- Proxy: inherited from abapGit's settings; this tool adds none.
- TLS: the git host's certificate chain must be trusted in the system's certificate store.
- The batch user needs authorisation for abapGit deserialisation and for the configured request.

## 5. Verification and failure handling

### The verifier

After every pull the verifier checks three independent things. The tag is recorded as deployed
only if **all three** hold.

1. **Deserialisation timestamp moved.** The repository's deserialisation timestamp in abapGit's
   persistence is read before and after the pull and must differ. A pull that reports success but
   changes nothing (for example because it ran on a stale session) is thereby detected.
2. **The requested tag is checked out.** The repository's current ref and commit equal the tag
   that was requested. This catches a pull that ran against the wrong ref or repository.
3. **Nothing left inactive.** No inactive objects remain in the repository's package, because a
   pull activates only what it creates.

### Failure classes

| Failure | Behaviour |
|---|---|
| Remote unreachable, authentication or TLS error | Log; state unchanged; retried next run. |
| Pull raises an error | Log the abapGit messages; state unchanged. |
| Pull reports success but the verifier fails | Treated as a failed deployment; log which check failed; state unchanged. |
| `REPO_KEY` unknown or config row invalid | Fail that repository with a clear message; others continue. |
| Locked objects or unusable transport request | Fail that repository; log the lock owner or the request problem. |

### Retry, noise, rollback, concurrency

- State moves only after verification, so a failed tag is retried on every run.
- The log records consecutive failures per repository and tag; alerts fire on the first failure
  and again after a configurable N consecutive failures.
- No automatic rollback in v1. If a tag fails part-way the system may be partly updated; recovery
  is the next tag, or re-running the same tag once the cause is fixed. The state table keeps the
  last good tag so rollback can be added later.
- An enqueue lock on the repository key prevents two runs, or a run and a manual pull, from
  deploying the same repository at once. A run that cannot obtain the lock skips that repository
  and logs it.

### Reporting

One application-log entry per repository per run: tag, outcome, failed check if any. The report
prints the same summary so the batch spool is readable without opening the log.

## 6. Testing

**Unit tests** (ABAP Unit; no network, no abapGit; fakes for every interface):

- newer tag → deploy called; state written only when the verifier passes
- same tag as state → nothing called
- verifier fails → state unchanged, failure logged, next repository still runs
- `dry-run` → nothing deployed, nothing written, plan logged
- one repository throws → the others still run
- tag selection uses version order

**Integration tests** (on a development system with a throwaway repository):

- real pull of a tag into a local package and into a transportable package with a standing request
- **silent no-op**: force a pull that changes nothing and assert the verifier rejects it
- private repository with a token, public repository without
- two repositories with overlapping names: the deployer must act on the one identified by key

## 7. Assumptions to verify before implementation

These are not yet confirmed. If any fails, this design changes and the spec is updated.

1. Which abapGit flavour is installed (standalone single program vs. developer edition) and which
   classes are therefore callable.
2. Whether a repository can be switched to a tag programmatically, and how.
3. The abapGit API for supplying a token in a non-dialog context.
4. The secure-store API available on supported releases.
5. Whether a supported API exposes the deserialisation timestamp, so persistence need not be read
   directly.
6. How to list a repository's remote tags through abapGit's git transport without a full fetch.
7. Behaviour differences between an older ECC-era release and a current ABAP platform release.

## 8. Repository layout

The repository is itself the abapGit repository for package `Z_CONTINUOUS_DEPLOYMENT`. ABAP
source lives under `src/` in abapGit format, generated by SAP's own serialisation and never
hand-written. Static analysis with abaplint runs in CI. Changes land through pull requests.
