# CLAUDE.md

Project-specific guidance for AI agents working in a repository created from this template. Read this alongside [README.md](README.md).

This file is part of the template and gets copied into every new project. Adjust it to your project's real constraints (package name, system, style) after you create the repo.

## What this repo is

A git repository that holds the ABAP source of **one** ABAP package. Code is written by AI agents (you), reviewed by humans in pull requests, and synchronized with a SAP system either:

- **via ADT** — you edit SAP directly through [`aibap.mcp`](https://github.com/Hochfrequenz/aibap.mcp) and pull the resulting state back into git for review, or
- **via abapGit** — you write files in this working tree, push to git, and abapGit pulls them into SAP.

Both workflows are supported. The README describes when each applies.

## Pick the right MCP for the job

Before you start editing, check which MCPs are actually available in this session (`/mcp` in Claude Code). You may have one, the other, or both.

| Task | Prefer |
|---|---|
| Read, write, activate, syntax-check, test ABAP code | `aibap.mcp` (ADT) |
| Create a new object (`PROG`, `CLAS`, `INTF`, `FUGR`, `MSAG` — all systems; `DDLS`, `TABL`, `DTEL`, `DOMA` — S/4HANA only) | `aibap.mcp` (`create_object`) |
| Transport management (create, assign, release) | `aibap.mcp` (`transport` group) |
| ATC / ABAP Unit / syntax check | `aibap.mcp` |
| Short dumps / ST22 analysis | `aibap.mcp` (`shortdumps` group) |
| Pulling a git branch into SAP via abapGit | `sapgui.mcp` (`sap_abapgit_pull`) — requires the `Z_ABAPGIT_PULL_MCP_SHORTCUT` report on SAP side |
| Running an arbitrary transaction, looking at a screen | `sapgui.mcp` (`sap_transaction`, browser tools) |
| Customizing / table maintenance screens | `sapgui.mcp` |

If a tool you need is not visible, tell the user — they may need to enable a tool group (e.g. `--tools=...` for `aibap.mcp`) or start the other MCP.

## Rules of the road

### Do not hand-write abapGit XML
If you are tempted to write a `*.clas.xml`, `*.prog.xml`, etc. from scratch: stop. Either

1. use `aibap.mcp`'s `create_object` to let SAP serialize the object and then pull the result into git via abapGit, or
2. copy the structure from an existing object of the same type in the same repo.

Broken abapGit XML will cause pull failures on SAP side. Never commit a state whose XML you are not sure abapGit will accept.

### File naming
Use **all lowercase** file names for ABAP workbench objects (e.g. `zcl_my_service.clas.abap`, not `ZCL_MY_SERVICE.clas.abap`). abapGit is case-sensitive on Linux/macOS checkouts.

### Clean ABAP style
Default to [Clean ABAP](https://github.com/SAP/styleguides/blob/main/clean-abap/CleanABAP.md), but ask the human about the house style first — see README → "Creating ABAP Code" for the rationale.

### Never commit secrets
`.mcp.json`, `opencode.json`, `systems.json`, PATs, passwords — none of these belong in the repo. The template's `.gitignore` already excludes `.mcp.json`, `opencode.json`, and Claude Code local settings.

### Respect the transport system
Every change lands in a transport request — ask the user which TR to target, and **never release a transport without explicit permission**. See README → "Transport Management" for the per-workflow details.

### Write reviewable commits
Keep commits reviewable: one logical change per commit, descriptive message, and a PR description that explains *why*. See README → "Review Culture" for why PR review matters in this project.

## When things go wrong

- **Syntax error on activate:** fix it before moving on. Don't commit code that doesn't activate.
- **423 lock errors via ADT:** the object is locked by someone else, or SAP is checking the lock against the wrong enqueue table. `aibap.mcp` auto-locks; if a persistent 423 remains on a fresh object you own, the fix is usually a stateful session (`X-sap-adt-sessiontype: stateful`) — check `aibap.mcp`'s own CLAUDE.md for current guidance.
- **abapGit pull fails:** most likely XML serialization issue (see "Do not hand-write abapGit XML" above) or a missing PAT. Read the error from `sap_abapgit_pull` or the status bar before guessing.
- **MCP server not visible:** the user may need to restart Claude Code after editing `.mcp.json` or env vars.

## Project-specific notes

Start with `docs/HANDOVER.md`: where things stand, what to do next, and what to ask the maintainer first.

- **What this is:** an ABAP batch job inside an SAP development system that polls git for new `vX.Y.Z` tags on
  abapGit repositories and deploys them with an abapGit pull. Spec: `docs/superpowers/specs/`. Plans:
  `docs/superpowers/plans/`.
- **Package:** `Z_CONTINUOUS_DEPLOYMENT` (customer namespace). Object prefix `CDEPLOY`; the naming scheme and the
  DDIC length limits are enforced by `abaplint.json`.
- **System(s):** an older and a newer system, development only. Their names are deliberately not in this
  (public) repository. Develop on the older one first (the language floor is ABAP 7.40 SP08), then pull into the
  newer one with abapGit and compare unit-test counts.
- **Style:** Clean ABAP, lower-case file names, keywords as SAP serialises them (do not fight the Class Builder).
- **Test coverage expectations:** every class in `src/` has ABAP Unit tests; logic is testable without abapGit,
  a database, the network or the clock (they sit behind interfaces).
- **Transport discipline:** ask which workbench request to use; pass the request, not the task; **never release
  a request** without explicit permission.
- **Pull requests:** squash merge only; a separate review agent runs at least once before a PR is marked ready.
- **Public repository:** no system names, hosts, clients, customer or person names, tokens, or names of private
  repositories anywhere.
